# 06 - SSH tunnels

Persistent SSH tunnels for accessing services running on remote nodes. Uses **autossh** to automatically reconnect dropped tunnels, with OpenSSH keepalive for health checks.

## Port schema convention

Assign each node a single-digit prefix. The local port is the prefix concatenated with the last 4 digits of the remote port. This keeps local ports predictable and avoids collisions across nodes.

| Node | Prefix | Local port range |
|------|--------|------------------|
| `<NODE_A>` | `1` | `1xxxx` |
| `<NODE_B>` | `2` | `2xxxx` |
| `<NODE_C>` | `3` | `3xxxx` |

**Worked example** with three services on `<NODE_C>` (prefix `3`):

| Remote port | Last 4 digits | Local port | Service |
|-------------|---------------|------------|---------|
| `8080` | `8080` | `38080` | Web app |
| `8000` | `8000` | `38000` | API server |
| `5432` | `5432` | `35432` | Database |

Access the web app at `http://localhost:38080` on your workstation.

## Placeholders

| Placeholder | Meaning |
|-------------|---------|
| `<NODE_NAME>` | Short alias for the node (e.g. `alpha`, `lab`) |
| `<NODE_IP>` | Node IP or Tailscale IP |
| `<USER>` | SSH user on the remote node |
| `<LOCAL_PORT>` | Port on your workstation (prefix + last 4 of remote) |
| `<REMOTE_PORT>` | Port the service listens on remotely |
| `<SERVICE_NAME>` | Human-readable service label for the comment |

---

## Install autossh

**Precondition:** macOS with Homebrew, or Ubuntu/Debian with apt.

**Action (macOS):**

```bash
brew install autossh
```

**Action (Linux):**

```bash
sudo apt install -y autossh
```

**Verify:**

```bash
autossh -V
```

---

## Configure SSH tunnel hosts

**Precondition:** autossh installed. SSH key authentication working for each node.

**Action:** add one block per node to `~/.ssh/config`:

```
Host tunnel-<NODE_NAME>
  HostName <NODE_IP>
  User <USER>
  LocalForward <LOCAL_PORT> 127.0.0.1:<REMOTE_PORT>    # <SERVICE_NAME>
  ServerAliveInterval 60
  ServerAliveCountMax 3
```

Add multiple `LocalForward` lines to tunnel several services through the same connection. Example with three services:

```
Host tunnel-lab
  HostName 100.x.y.z
  User deploy
  LocalForward 38080 127.0.0.1:8080    # web-app
  LocalForward 38000 127.0.0.1:8000    # api-server
  LocalForward 35432 127.0.0.1:5432    # postgres
  ServerAliveInterval 60
  ServerAliveCountMax 3
```

**Verify:**

```bash
ssh -N tunnel-<NODE_NAME>
```

If the connection stays open with no errors, forwarding is working. Press `Ctrl+C` to close.

---

## Shell aliases for tunnel management

**Precondition:** SSH config blocks in place.

**Action:** add to your shell rc file (`.zshrc`, `.bashrc`, or a sourced file):

```bash
alias tunnels-up='autossh -M 0 -fN tunnel-<NODE_A> && autossh -M 0 -fN tunnel-<NODE_B>'
alias tunnels-down='pkill -f autossh'
alias tunnels-status='ps aux | grep "[a]utossh"'
```

**Verify:**

```bash
tunnels-up
tunnels-status
```

Expected output: one `autossh` process per node. Access a forwarded service (e.g. `curl -s http://localhost:38080`) to confirm traffic flows.

---

## Flag reference

| Flag | Purpose |
|------|---------|
| `-M 0` | Disable autossh monitoring port. Relies on OpenSSH `ServerAliveInterval` / `ServerAliveCountMax` for keepalive instead. |
| `-f` | Fork to background after connection is established. |
| `-N` | No remote command execution, forwarding only. |

---

## Persistence at login

Tunnels started with `tunnels-up` survive until the machine reboots or the process is killed.

For automatic startup at login:

- **macOS:** create a launchd plist in `~/Library/LaunchAgents/` that runs the autossh commands.
- **Linux:** create a systemd user service (`~/.config/systemd/user/`) with `WantedBy=default.target`.
