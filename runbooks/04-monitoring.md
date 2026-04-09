# 04 - Monitoring

Two lightweight tools for observability on a single server: **Netdata** for system metrics and **Dozzle** for Docker log viewing. Both run as Docker containers, bind to localhost only, and are accessed via SSH tunnel or Tailscale.

## Placeholders

| Placeholder | Description | Example |
|---|---|---|
| `<HOSTNAME>` | Server hostname | `myserver` |
| `<USER>` | SSH user on the server | `deploy` |
| `<IP_ADDRESS>` | Server IP address | `203.0.113.10` |

## 1. Netdata (system metrics)

Real-time monitoring dashboard: CPU, RAM, disk, network, Docker containers. 800+ metrics out of the box, roughly 30 MB RAM footprint.

### 1.1 Create directory

**Precondition:** Docker and Docker Compose are installed.

**Action:**

```bash
sudo mkdir -p /srv/docker/netdata
```

**Verify:**

```bash
ls -ld /srv/docker/netdata
```

Expected: directory exists, owned by root.

### 1.2 Write compose file

**Precondition:** `/srv/docker/netdata/` exists.

**Action:** Create `/srv/docker/netdata/docker-compose.yml`:

```yaml
version: "3.8"

services:
  netdata:
    image: netdata/netdata:stable
    container_name: netdata
    restart: unless-stopped
    hostname: <HOSTNAME>
    cap_add:
      - SYS_PTRACE
      - SYS_ADMIN
    security_opt:
      - apparmor:unconfined
    volumes:
      - ./netdata-config:/etc/netdata
      - netdata-lib:/var/lib/netdata
      - netdata-cache:/var/cache/netdata
      - /etc/passwd:/host/etc/passwd:ro
      - /etc/group:/host/etc/group:ro
      - /etc/localtime:/etc/localtime:ro
      - /proc:/host/proc:ro
      - /sys:/host/sys:ro
      - /var/run/docker.sock:/var/run/docker.sock:ro
    ports:
      - "127.0.0.1:19999:19999"

volumes:
  netdata-lib:
  netdata-cache:
```

> The Docker socket mount (`/var/run/docker.sock`) gives Netdata visibility into running containers. The host mounts (`/proc`, `/sys`, `/etc/passwd`, `/etc/group`) let it read system-level metrics. `SYS_PTRACE` and `SYS_ADMIN` capabilities are required for per-process monitoring and cgroup access.

### 1.3 Start Netdata

**Precondition:** Compose file is in place with `<HOSTNAME>` replaced.

**Action:**

```bash
cd /srv/docker/netdata && sudo docker compose up -d
```

**Verify:**

```bash
sudo docker ps --filter name=netdata --format '{{.Status}}'
```

Expected: `Up` with uptime.

```bash
curl -s -o /dev/null -w '%{http_code}' http://127.0.0.1:19999
```

Expected: `200`.

### 1.4 Access the dashboard

Netdata binds to 127.0.0.1 only. Access it via SSH tunnel from your local machine:

```bash
ssh -L 19999:127.0.0.1:19999 <USER>@<IP_ADDRESS>
```

Then open http://localhost:19999 in your browser.

If you have Tailscale set up (see [05-tailscale.md](05-tailscale.md)), you can also access it directly at `http://<TAILSCALE_IP>:19999` after opening port 19999 in UFW for the Tailscale subnet.

## 2. Dozzle (Docker log viewer)

Real-time Docker log viewer with a web UI. Zero storage, zero config, only shows live logs from running containers.

### 2.1 Create directory

**Precondition:** Docker and Docker Compose are installed.

**Action:**

```bash
sudo mkdir -p /srv/docker/dozzle
```

**Verify:**

```bash
ls -ld /srv/docker/dozzle
```

Expected: directory exists, owned by root.

### 2.2 Write compose file

**Precondition:** `/srv/docker/dozzle/` exists.

**Action:** Create `/srv/docker/dozzle/docker-compose.yml`:

```yaml
version: "3.8"

services:
  dozzle:
    image: amir20/dozzle:latest
    container_name: dozzle
    restart: unless-stopped
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock:ro
    ports:
      - "127.0.0.1:8080:8080"
    environment:
      - TZ=Europe/Rome
```

> Change `TZ` to your timezone if not in Europe/Rome. Dozzle uses it for log timestamps.

### 2.3 Start Dozzle

**Precondition:** Compose file is in place.

**Action:**

```bash
cd /srv/docker/dozzle && sudo docker compose up -d
```

**Verify:**

```bash
sudo docker ps --filter name=dozzle --format '{{.Status}}'
```

Expected: `Up` with uptime.

```bash
curl -s -o /dev/null -w '%{http_code}' http://127.0.0.1:8080
```

Expected: `200` or `301`.

### 2.4 Access the log viewer

SSH tunnel from your local machine:

```bash
ssh -L 8080:127.0.0.1:8080 <USER>@<IP_ADDRESS>
```

Then open http://localhost:8080 in your browser.

## Alternatives

| Tool | Use case | Notes |
|---|---|---|
| **Beszel** | Ultra-lightweight monitoring with UI | Agent uses 6 MB RAM, collector 23 MB. Good alternative if Netdata feels heavy. |
| **ctop** | TUI for live Docker container metrics | `docker run --rm -ti --name ctop -v /var/run/docker.sock:/var/run/docker.sock quay.io/vektorlab/ctop:latest` |
| **Loki + Grafana** | Log aggregation with retention and search | Overkill for a single server. Consider when you need structured alerting or multi-node log collection. |
