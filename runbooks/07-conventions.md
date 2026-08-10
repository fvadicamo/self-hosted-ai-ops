# 07 - Conventions

Infrastructure conventions adopted across all nodes. This is a reference document, not a step-by-step setup guide. Consult it when deciding where to place files, how to structure Docker services, or how to isolate dependencies.

---

## Filesystem hierarchy standard (FHS 3.0)

The official Linux Foundation standard defines where things go on the filesystem. Ubuntu follows it entirely. Sticking to FHS means any sysadmin can navigate your server without a map.

| Path | Purpose | Examples |
|---|---|---|
| `/opt/<name>/` | Self-contained pre-built applications installed externally | ComfyUI native install, third-party tools |
| `/usr/local/src/<name>/` | Sources maintained locally by the sysadmin (git clone for build) | Build context for `docker build` |
| `/usr/local/bin/` | Locally installed binaries (not from the package manager) | Custom system scripts |
| `/srv/` | Data served by running services (runtime data) | `/srv/docker/`, `/srv/myapp/` |
| `/etc/` | Static system configuration | `/etc/environment`, `/etc/systemd/` |
| `/var/` | Variable data: logs, spool, system cache | `/var/log/`, `/var/lib/docker/` |
| Home dirs | Only dotfiles, personal scripts, lightweight git repos | `.bashrc`, `.ssh/`, small projects |

**Key rules:**

- Home directories never contain: AI models, tool caches, processing output, system-level installations.
- `/opt/` is for pre-built executables, not for sources to compile or Docker configs.
- `/usr/local/src/` is the correct place for a repo cloned only to run `docker build`.

### What does NOT go where

| Wrong location | What was placed there | Correct location |
|---|---|---|
| `~/models/` | Large AI model files | Dedicated data mount or `/srv/` |
| `~/tool-cache/` | Build or inference cache | Dedicated scratch disk or `/var/cache/` |
| `/opt/myapp/` | A git repo cloned to build a Docker image | `/usr/local/src/myapp/` |
| `/srv/docker/myapp/` | Application source code | `/usr/local/src/myapp/` or `/opt/myapp/` |

---

## `/srv/docker/` convention

Standard structure for all Docker containers. Every service gets its own directory under `/srv/docker/`.

```
/srv/docker/
└── <service-name>/
    ├── <name>.env          # credentials and sensitive variables (chmod 600, root:root)
    ├── docker-compose.yml  # if using Compose
    ├── extra_config.yaml   # service-specific config (chmod 640)
    └── ...
```

### Permissions

```bash
# create the service directory
sudo mkdir -p /srv/docker/<name>
sudo chmod 700 /srv/docker/<name>    # only root can access (contains credentials)
sudo chown root:root /srv/docker/<name>

# credential files: even more restrictive
sudo chmod 600 /srv/docker/<name>/<name>.env
```

### What goes here

Configuration files and credentials for the container. Small, text-based config that the container needs at startup.

### What does NOT go here

- Application source code (goes in `/usr/local/src/` or `/opt/`)
- Large data files, model weights, media (use bind mounts from dedicated data paths)
- Build context or Dockerfiles for custom images (goes in `/usr/local/src/<name>/`)

---

## Docker bridge and UFW: known issue

Docker in bridge mode (`-p HOST:CONTAINER`) writes iptables rules directly, bypassing UFW. Any port mapped with `-p` becomes accessible from the network regardless of UFW rules.

The mechanism, because knowing it tells you which fixes can work. Docker DNATs published
ports in `nat/PREROUTING`, so the traffic takes the FORWARD path and never reaches INPUT,
where UFW's port rules live. In FORWARD, the jump to Docker's own chains comes before any
`ufw-*` chain:

```
-P FORWARD DROP
-A FORWARD -j DOCKER-USER        <- the only hook evaluated before Docker
-A FORWARD -j DOCKER-FORWARD     <- Docker accepts here
-A FORWARD -j ufw-before-forward <- UFW arrives after, and sees nothing
```

**A UFW rule on a bridge-published port is not "not enough": it is inert.** Worse, it reads
as a control that exists. If you find such rules, remove them rather than leaving them as
documentation of an intent nothing enforces.

Three fixes, in order of how surgical they are:

**1. Put the address in the publish spec.** The DNAT rule then carries a destination match,
so the port is genuinely scoped. Best when you control the compose file:

```bash
docker run -p 127.0.0.1:8000:8000 ...     # loopback only
docker run -p <VPN_IP>:8000:8000 ...      # one interface only
```

Caveat: binding a VPN or overlay address creates a start-order dependency on that interface
being up. For a service that starts at boot, order the unit after the VPN daemon.

**2. One rule in `DOCKER-USER`**, which Docker guarantees not to overwrite. Best when several
services share the same policy, or when you cannot recreate the containers:

```bash
# deny forwarding from an untrusted interface to any container
iptables -I DOCKER-USER -i <IFACE> -m conntrack --ctstate NEW -j DROP
```

`--ctstate NEW` is mandatory, not cosmetic: return traffic of the containers' own outbound
connections arrives on the same interface, so without it you break every `docker pull`.
Apply it to `ip6tables` too: if Docker IPv6 is enabled later, IPv6 DNAT appears and an
IPv4-only rule stops covering it, silently. The rule is not persistent by itself, so drive it
from a systemd unit ordered after (and `PartOf=`) the Docker service.

**3. `--network=host`.** The container shares the host network stack and UFW applies normally.
Simple, but it removes network isolation and is not available for multi-container stacks that
rely on an internal bridge network:

```bash
docker run --network=host ...  # UFW applies, port controlled by ufw rules
# the service binds directly on the host; use --listen/--bind in the process inside
```

**How to verify, without fooling yourself.** Do not read the firewall rules: source traffic
from an address the firewall does not allow and see whether it answers. A throwaway container
on a different bridge works. Watch out for the blind spot: if the source sits on the *same*
bridge as the target, the DNAT rule does not match (`! -i br-X`) and the port looks blocked
when it is not.

### When to use `--network=host`

24/7 system services on dedicated machines where Docker network isolation adds no real value. The service binds directly on the host, and UFW controls access.

### When to use bridge (`-p`)

- Multi-tenant VPS where multiple services run for different users
- Environments with multiple containers that need Docker-internal networking
- Anywhere network isolation between containers matters

---

## Environment isolation: fundamental rule

**Never install application dependencies in the global system or user environment.** This rule applies to every node - VPS, workstations, any new machine.

### Hierarchy of solutions (in order of preference)

1. **Stdlib only** - if the script uses only standard library modules, there are no dependencies to manage. Always prefer this approach.
2. **Python venv in `/opt/`** - for Python scripts with external dependencies: create a venv in `/opt/<name>/venv`. Never use global pip or `~/.local`.
3. **Docker** - for multi-dependency services, daemons, or tools without a native isolation mechanism. The container carries its entire environment.
4. **nvm** - for Node.js in personal development environments. Use nvm to install node per-project. Exception: services that need systemd compatibility may require system Node (via NodeSource apt), since systemd units do not load shell init and cannot resolve nvm paths.

### Anti-patterns

```bash
# FORBIDDEN: pollutes global/user pip
pip install requests
pip install --user requests

# FORBIDDEN: sudo npm install -g pollutes system node_modules
sudo npm install -g something

# FORBIDDEN: apt install for application dependencies
sudo apt install python3-requests   # only for system tools, not for custom scripts
```

### Correct venv pattern

```bash
# create an isolated environment in /opt/ (never in home directory)
sudo python3 -m venv /opt/<name>/venv
sudo /opt/<name>/venv/bin/pip install <dependency>

# in cron/systemd, always use the absolute path to the venv interpreter
/opt/<name>/venv/bin/python /usr/local/bin/<script>.py
```

### Fallback: Docker

If a tool requires many dependencies or a specific runtime environment and there is no reasonable stdlib/venv alternative, use Docker. The Dockerfile and config go in `/srv/docker/<name>/` following the standard convention above.

---

## Localhost-only binding

All services listen on `127.0.0.1`, never on `0.0.0.0`. External access goes through SSH tunnels or a reverse proxy.

```bash
# correct: bound to localhost only
-p 127.0.0.1:8000:8000

# wrong: accessible from any interface
-p 8000:8000
```

Inside the container, `--listen 0.0.0.0` is fine because the container network is isolated. The host-side binding is controlled by the `-p` flag, not by the process inside.

This means:

- From the host itself: `curl http://127.0.0.1:8000` works.
- From the network: connection refused (unless UFW/iptables explicitly allows it via other means).
- Remote access: use an SSH tunnel (`ssh -L 8000:127.0.0.1:8000 user@host`) or configure a reverse proxy that terminates TLS and forwards to localhost.
