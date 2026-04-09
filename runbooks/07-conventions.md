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

**Solution for services that must respect UFW:** use `--network=host`. The container shares the host network stack, and UFW rules apply normally.

```bash
# WRONG for services that must respect UFW:
docker run -p 8000:8000 ...    # bypasses UFW, port accessible to everyone

# CORRECT:
docker run --network=host ...  # UFW applies, port controlled by ufw rules
# the service binds directly on the host; use --listen/--bind in the process inside
```

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
