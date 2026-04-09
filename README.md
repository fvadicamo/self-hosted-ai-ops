# Self-hosted AI ops

> Opinionated dev-environment and networking runbooks for self-hosted infrastructure. Includes LLM-executable runbooks for AI assistants.

Set up a complete development environment on a remote Linux server: shell, dev tools, tmux, monitoring, Tailscale mesh networking, SSH tunnels, and infrastructure conventions. Works as a standalone guide or as a companion to [self-hosted-ai-lab](https://github.com/fvadicamo/self-hosted-ai-lab) and [self-hosted-ai-rig](https://github.com/fvadicamo/self-hosted-ai-rig).

Every runbook follows a Precondition / Action / Verify pattern designed for [Claude Code](https://claude.ai/code) or any AI coding assistant. Give a runbook to your assistant and it will execute the setup for you. Also works perfectly fine as a guide for humans.

## What you build

```
Your remote Linux server
├── Shell: zsh + antidote + starship + modern CLI tools
├── Dev tools: ghq, gh CLI, nvm, Claude Code
├── Git: SSH key auth + commit signing
├── tmux: persistent sessions, split panes, mouse support
├── Monitoring: Netdata (metrics) + Dozzle (container logs)
├── Networking: Tailscale mesh VPN + ACL + node sharing
├── SSH tunnels: autossh, persistent forwarding
└── Conventions: FHS, /srv/docker/, environment isolation
```

## Who is this for

- You run **remote Linux servers** (VPS, workstations, homelabs) and want a consistent, well-documented setup
- You want a **productive development environment** on headless servers: modern shell, persistent sessions, fast navigation
- You need **secure remote access** to services without exposing ports to the internet
- You want something you can hand to an AI assistant and say "set this up for me"

## Requirements

- **Ubuntu 24.04 LTS** (server or desktop, headless recommended)
- **SSH access** from your local machine
- A **GitHub account** (for gh CLI and commit signing)
- A **Tailscale account** (free tier works, for runbooks 05-06)

## Quick start

Follow the runbooks in order. Each one is self-contained - you can stop at any step and have a working setup.

| # | Runbook | What it does | Source |
|---|---------|-------------|--------|
| 01 | [Shell](runbooks/01-shell.md) | zsh, antidote, starship, modern CLI tools (eza, bat, fzf, ripgrep, zoxide) | Terminal foundation |
| 02 | [Dev tools](runbooks/02-devtools.md) | ghq, gh CLI, nvm, Node.js, Claude Code, git config, SSH commit signing | Development workflow |
| 03 | [tmux](runbooks/03-tmux.md) | Persistent sessions, split panes, mouse support, custom keybindings | Remote session management |
| 04 | [Monitoring](runbooks/04-monitoring.md) | Netdata (system metrics) + Dozzle (Docker log viewer) | Observability |
| 05 | [Tailscale](runbooks/05-tailscale.md) | Mesh VPN, ACL policies, node sharing, VPS-as-front-door pattern | Secure networking |
| 06 | [SSH tunnels](runbooks/06-ssh-tunnels.md) | autossh, SSH config, port schema, persistent forwarding | Service access |
| 07 | [Conventions](runbooks/07-conventions.md) | FHS 3.0, /srv/docker/, Docker+UFW bypass, environment isolation, localhost binding | Infrastructure standards |
| 08 | [Remote workflow](runbooks/08-remote-workflow.md) | Repo management, SSH client config, remote editor, mobile access | Daily workflow |

Runbooks 01-03 give you a productive remote shell. Add 04 for monitoring, 05-06 for networking, 07 for conventions, 08 for daily workflow.

## Runbook format

Every runbook follows this structure:

```
## Step name

Precondition: what must be true before this step
Action: exact commands to run
Verify: command + expected output to confirm success

> Note: optional context for humans (why this choice was made)
```

Commands use `<PLACEHOLDER>` format for values you must customize. All placeholders are listed at the top of each runbook.

## Related projects

| Project | What it covers |
|---------|---------------|
| [self-hosted-ai-lab](https://github.com/fvadicamo/self-hosted-ai-lab) | VPS provisioning, hardening, Docker, Caddy, n8n, OpenClaw |
| **self-hosted-ai-ops** (this repo) | Dev environment, networking, conventions |
| [self-hosted-ai-rig](https://github.com/fvadicamo/self-hosted-ai-rig) | GPU workstation: NVIDIA drivers, vLLM, Open WebUI, model catalog |

The three repos are complementary. ai-lab gives you a hardened VPS. ai-ops gives you the dev environment and networking layer. ai-rig gives you a local AI inference stack.

## For AI assistants

If you are an AI coding assistant executing these runbooks:

- Read the full runbook before starting
- Execute steps sequentially - each depends on the previous
- Always run the "Verify" step and check output matches expected
- If a verify step fails, stop and diagnose before continuing
- Placeholders (`<VALUE>`) must be resolved before execution
- Never skip verify steps, even if the command appeared to succeed
- After completing a runbook, report which steps succeeded and which need attention

## License

MIT

## Contributing

Issues and PRs welcome.
