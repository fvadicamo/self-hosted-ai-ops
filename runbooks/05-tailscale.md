# 05 - Tailscale

Tailscale is a mesh VPN built on WireGuard. Each device runs a lightweight agent that connects to a coordination server, which distributes keys and configuration. Devices talk directly to each other (peer-to-peer) whenever possible, falling back to relay servers (DERPs) when NAT prevents direct connections. The result is a flat, private network across all your machines, no matter where they are.

This runbook covers installation, mesh topology, ACL policy, node sharing, and architectural patterns for combining Tailscale with public VPS nodes.

## Placeholders

| Placeholder | Description | Example |
|---|---|---|
| `<USER>` | SSH user on the server | `deploy` |
| `<EMAIL>` | Tailscale account email | `admin@example.com` |
| `<TAILSCALE_IP>` | Tailscale IP (100.x.y.z) assigned to a node | `100.64.0.1` |
| `<VPS_IP>` | Public IP of a VPS | `203.0.113.10` |
| `<SUBNET>` | Local subnet to advertise | `10.0.10.0/24` |

---

## 1. Installation

### 1.1 Linux server

**Precondition:** Ubuntu/Debian server with sudo access.

**Action:**

```bash
curl -fsSL https://tailscale.com/install.sh | sh
sudo tailscale up
```

The second command opens a browser URL for authentication. On headless servers, copy the URL and open it on another machine. After authentication, the node appears in the admin console.

**Verify:**

```bash
tailscale status
```

Expected: the node shows as connected, with a `100.x.y.z` IP assigned.

Record the assigned IP. You will need it for ACL configuration.

### 1.2 macOS client

**Precondition:** macOS with admin access.

**Action:** Download the client from [tailscale.com/download](https://tailscale.com/download). Install and sign in with the same account used for the servers.

**Verify:** The menu bar icon shows "Connected" and all enrolled nodes appear in the device list.

### 1.3 Subnet router

A subnet router advertises a local LAN to the Tailscale mesh, making devices on that LAN reachable without installing Tailscale on each one.

**Precondition:** Tailscale installed on the gateway device (e.g., a router or dedicated machine on the target LAN). IP forwarding enabled.

**Action:**

```bash
# enable IP forwarding
echo 'net.ipv4.ip_forward = 1' | sudo tee /etc/sysctl.d/99-tailscale.conf
sudo sysctl -p /etc/sysctl.d/99-tailscale.conf

# advertise the subnet
sudo tailscale up --advertise-routes=<SUBNET>
```

Then approve the route in the admin console: Machines > select the subnet router > Edit route settings > approve the advertised route.

**Verify:**

```bash
tailscale status
```

Expected: the subnet router shows the advertised route. From another Tailscale node, you can ping devices on `<SUBNET>`.

---

## 2. Mesh topology and node roles

A typical self-hosted AI setup has several node roles:

| Role | Description | Example |
|---|---|---|
| Client admin | Your laptop or desktop, full access to everything | `laptop` |
| Client mobile | Phone, same access level as admin client | `phone` |
| GPU server | Private backend, never exposed to internet | `gpu-server` |
| VPS | Public-facing server, tagged `tag:vps` | `vps-1` |
| Exit node | Routes internet traffic through a specific country | `exit-ch` |
| Subnet router | Advertises a local LAN to the mesh | `lan-gateway` |

Node roles are enforced through tags and ACL policy (next section), not by Tailscale itself. The mesh is flat by default, so without ACL rules every node can reach every other node.

---

## 3. ACL policy

Tailscale ACLs use a default-deny model. You define explicit grants to allow traffic. The policy is managed in the admin console under Access Controls, or via the `tailscale` CLI with `--policy-file`.

The example below uses the `grants` format (Tailscale's newer, recommended format). All grants are allow rules; there is no explicit deny.

### 3.1 Template ACL

```json
{
    "tagOwners": {
        "tag:vps":      ["<EMAIL>"],
        "tag:exitnode":  ["<EMAIL>"],
        "tag:iot":       ["<EMAIL>"]
    },
    "groups": {
        "group:admin": ["<EMAIL>"]
    },
    "hosts": {
        "gpu-server":  "<TAILSCALE_IP>",
        "vps-1":       "<TAILSCALE_IP>",
        "vps-2":       "<TAILSCALE_IP>",
        "exit-ch":     "<TAILSCALE_IP>"
    },
    "autoApprovers": {
        "routes": {
            "<SUBNET>": ["<EMAIL>"]
        },
        "exitNode": ["tag:exitnode"]
    },
    "grants": [
        // admin: full access to everything
        {
            "src": ["group:admin"],
            "dst": ["*"],
            "ip":  ["*"]
        },

        // VPS nodes: can only reach the inference API on the GPU server
        {
            "src": ["tag:vps"],
            "dst": ["gpu-server"],
            "ip":  ["tcp:8000"]
        },

        // collaborators (via node sharing): SSH + selected services on GPU server
        {
            "src": ["collaborator@example.com"],
            "dst": ["gpu-server"],
            "ip":  ["tcp:22", "tcp:8000", "tcp:8080"]
        },

        // IoT devices: inference API only
        {
            "src": ["tag:iot"],
            "dst": ["gpu-server"],
            "ip":  ["tcp:8000"]
        }

        // exit nodes: no grants - isolated from all internal nodes
        // they only serve as exit points for outbound internet traffic
    ],
    "tests": [
        // admin reaches everything
        {
            "src":    "<EMAIL>",
            "accept": ["gpu-server:22", "gpu-server:8000", "vps-1:22", "vps-2:22"]
        },
        // VPS reaches only inference API
        {
            "src":    "tag:vps",
            "accept": ["gpu-server:8000"],
            "deny":   ["gpu-server:22", "vps-1:22", "vps-2:22"]
        },
        // collaborator reaches only allowed ports on GPU server
        {
            "src":    "collaborator@example.com",
            "accept": ["gpu-server:22", "gpu-server:8000", "gpu-server:8080"],
            "deny":   ["vps-1:22", "vps-2:22"]
        },
        // IoT reaches only inference API
        {
            "src":    "tag:iot",
            "accept": ["gpu-server:8000"],
            "deny":   ["gpu-server:22", "vps-1:22"]
        },
        // exit node cannot reach internal nodes
        {
            "src":  "tag:exitnode",
            "deny": ["gpu-server:22", "gpu-server:8000", "vps-1:22"]
        }
    ]
}
```

### 3.2 Key concepts

**tagOwners:** defines which users can assign a tag to a machine. Tags are used in grants as `src` or `dst` selectors.

**groups:** named sets of users. `group:admin` is a common pattern for the infrastructure owner.

**hosts:** aliases for Tailscale IPs. Makes the policy readable and avoids hardcoding IPs in grants. Note: using IP addresses directly in `dst` is confirmed to work; alias support in `dst` may vary, so test after applying.

**autoApprovers:** automatically approves subnet routes and exit nodes without manual confirmation in the admin console. Useful to avoid re-approving after a node restart.

**grants:** each grant allows traffic from `src` to `dst` on the specified `ip` (protocol:port). Omitting `ip` or using `["*"]` allows all ports.

**tests:** declarative assertions that the policy engine evaluates when you save the policy. If a test fails, the policy is rejected. Always write tests for every role.

---

## 4. Node sharing

Node sharing gives collaborators access to specific machines without consuming user slots on the free plan.

### How it works

- The owner shares a specific node to the collaborator's personal tailnet
- The collaborator has their own free Tailscale account
- The shared node appears in the collaborator's device list, reachable via its Tailscale IP
- The collaborator does **not** count as a user in the owner's tailnet
- The collaborator can only see the shared node, not any other node in the owner's mesh
- Sharing is unidirectional: the collaborator sees the shared node, but the owner cannot see the collaborator's devices
- The owner's ACL policy does **not** apply to traffic from shared nodes. Protection is delegated to the host firewall (UFW) on the shared machine

### Management

Share a node: admin console (login.tailscale.com/admin/machines) > select machine > Share. Enter the collaborator's Tailscale email or tailnet name.

Revoke access: same menu, remove the share.

### When to use node sharing vs adding users

| Scenario | Approach |
|---|---|
| 1-2 collaborators who need access to a single server | Node sharing (no user slots consumed) |
| Team members who need access to multiple nodes | Add as users in the tailnet |
| External contributors who should never see internal infra | Node sharing (they see only the shared node) |

### Firewall considerations

Since ACLs do not apply to shared-node traffic, the host firewall is the only access control layer. Configure UFW on the shared machine to restrict which ports are reachable:

```bash
# allow SSH and specific services from Tailscale range
sudo ufw allow from 100.64.0.0/10 to any port 22 comment "SSH - Tailscale"
sudo ufw allow from 100.64.0.0/10 to any port 8000 comment "Inference API - Tailscale"
sudo ufw allow from 100.64.0.0/10 to any port 8080 comment "Web UI - Tailscale"
```

---

## 5. VPS as front door pattern

The GPU server or homelab should never be exposed directly to the internet. A VPS acts as the public layer: it receives external traffic, validates it (SSL termination, authentication, rate limiting), and calls the private backend via Tailscale for GPU processing.

```
Internet
   |
   v
VPS (Caddy/Nginx, SSL, rate limiting, auth)    <- public IP: <VPS_IP>
   | Tailscale (e.g., tcp:8000)
   v
GPU server (vLLM, ComfyUI, ...)                <- no public IP, only Tailscale
```

### Why this matters

- The GPU server has no open ports on the public internet. Attackable only if a Tailscale node is compromised.
- The VPS handles TLS certificates, DDoS mitigation, and user authentication. The GPU server never sees the end user's identity.
- ACL policy restricts VPS access to only the inference port (e.g., `tcp:8000`). Even if the VPS is compromised, the attacker cannot SSH into the GPU server or reach other nodes.
- Multiple VPS nodes can point to the same GPU backend, enabling different frontends (web app, API, chat UI) without duplicating GPU resources.

### Example: reverse proxy on VPS

A minimal Caddy configuration on the VPS, forwarding authenticated requests to the GPU server via Tailscale:

```
api.example.com {
    basicauth {
        <USER> <HASHED_PASSWORD>
    }
    reverse_proxy <TAILSCALE_IP>:8000
}
```

The VPS resolves `<TAILSCALE_IP>` via the Tailscale interface. No public port needs to be opened on the GPU server.

---

## 6. Advanced features to evaluate

These features are not required for a basic setup but are worth evaluating as the infrastructure grows.

### Tailscale SSH

Manages SSH authentication via Tailscale identity instead of traditional SSH keys. Users authenticate through their Tailscale account, and the coordination server handles key distribution.

- Simplifies onboarding: no SSH keys to distribute or revoke
- Integrates with ACL: SSH access follows the same policy as other traffic
- Trade-off: adds dependency on Tailscale for SSH access. If Tailscale is down, you lose SSH (keep a fallback key-based access path)

### Tailscale Funnel

Exposes a local service to the public internet via Tailscale's infrastructure. The connection is outbound-only from your server, so no ports need to be opened.

- Requires `nodeAttrs` configuration in the policy file:
  ```json
  "nodeAttrs": [{"target": ["*"], "attr": ["funnel"]}]
  ```
- Alternative to running your own reverse proxy (Caddy/Nginx) on a VPS
- Useful for quick demos or services that do not justify a dedicated VPS
- Trade-off: traffic routes through Tailscale's servers, adding latency and a dependency

### Postures (device compliance)

Restricts access based on device attributes: minimum Tailscale version, OS version, disk encryption status.

- Useful when the access surface grows (many devices, multiple users)
- Overhead is excessive for small setups (fewer than 10 devices)
- Evaluate when adding IoT devices or untrusted machines to the mesh

---

## 7. VPN on smartphone: limitations

Tailscale is a VPN at the OS level. On iOS and Android, **only one VPN can be active at a time**. Activating a corporate VPN disconnects Tailscale, and vice versa.

### Implications

- If you need to reach a Tailscale service from your phone while a corporate VPN is active: not possible. The two VPNs cannot coexist.
- Tailscale's "split DNS" mode on mobile does not solve this. The single-VPN limitation is enforced by the OS, not by Tailscale.

### Workaround: Cloudflare Tunnel

For services that must be reachable from mobile without depending on Tailscale (e.g., a home automation dashboard), Cloudflare Tunnel is an alternative:

- Creates an outbound connection from the server to Cloudflare, no inbound ports needed
- Exposes the service on a public HTTPS URL with authentication
- Reachable from any browser, regardless of active VPN

This is the primary scenario where Cloudflare Tunnel has a concrete advantage over Tailscale-only access.

| Approach | Pro | Con | Best for |
|---|---|---|---|
| Tailscale only | Private, zero public exposure | Conflicts with corporate VPN on mobile | Personal/crew access from trusted devices |
| Cloudflare Tunnel | No port opening, DDoS protection, works alongside corporate VPN | Dependency on Cloudflare, public URL | Services that need mobile access without VPN |
| Public IP + Caddy/Nginx | Full control, no external dependency | IP exposed, certificate management | VPS with stable public services |
