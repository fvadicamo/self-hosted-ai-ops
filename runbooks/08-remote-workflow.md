# 08 - Remote workflow

Daily workflow patterns for working on a remote Linux server: creating repos, client-side setup, tmux session management, editor integration, and mobile access.

Assumes [01-shell.md](01-shell.md) through [03-tmux.md](03-tmux.md) are completed (shell, dev tools, ghq, git, tmux all configured on the server).

## Placeholders

| Placeholder | Description | Example |
|-------------|-------------|---------|
| `<SERVER_ALIAS>` | SSH config alias for the remote server | `myserver` |
| `<SERVER_IP>` | IP address of the remote server (or Tailscale IP) | `100.64.1.50` |
| `<ORG>` | GitHub organization or username | `myorg` |
| `<REPO_NAME>` | Repository name | `my-project` |
| `<USER>` | Your Linux username on the server | `jane` |

---

## 8.1 Creating a new repo on the server

Run these on the remote server. The directory structure follows the ghq convention (`~/code/github.com/<org>/<repo>`).

**Action:**

```bash
mkdir -p ~/code/github.com/<ORG>/<REPO_NAME>
cd ~/code/github.com/<ORG>/<REPO_NAME>

# create .gitignore
cat > .gitignore << 'EOF'
.DS_Store
*.bak-*
.claude/
EOF

git init
git add .
git commit -m 'chore: initial commit'

# create the repo on GitHub and push
gh repo create <ORG>/<REPO_NAME> --private --description '<description>'
git remote add origin git@github.com:<ORG>/<REPO_NAME>.git
git push -u origin main
```

> If your remote URLs use HTTPS, switch them to SSH: `git remote set-url origin git@github.com:<ORG>/<REPO_NAME>.git`.

---

## 8.2 Client-side setup

These steps are for your local machine (laptop/desktop) to mirror the same repo structure and connect to the server efficiently.

### Install ghq

ghq keeps local repos in the same directory layout as on the server.

```bash
# macOS
brew install ghq

# Linux
# see runbook 02 for manual install
```

Configure the root directory:

```bash
mkdir -p ~/code
git config --global ghq.root ~/code
```

### Clone existing repos

```bash
ghq get github.com/<ORG>/<REPO_NAME>
```

ghq places repos automatically under `~/code/github.com/<ORG>/<REPO_NAME>`.

### Shell aliases for remote connection

Add these to your shell config (e.g., `~/.zshrc` or a sourced file):

```bash
# connect to the server via tmux
# -A: attach if session "main" exists, create it otherwise
alias k='ssh -t <SERVER_ALIAS> "tmux new-session -A -s main"'
alias ks='ssh <SERVER_ALIAS> "tmux ls 2>/dev/null || echo no active sessions"'
```

### Repo navigation function

If you have ghq and fzf on the client too, add this function for fast repo switching:

```bash
# navigate repos with ghq + fzf
# accepts an optional argument as a pre-filter
j() {
  local dir
  dir=$(ghq list -p | fzf -q "${1:-}" --select-1 --exit-0)
  [[ -n "$dir" ]] && cd "$dir"
}
```

Usage: `j` opens the interactive list; `j infra` pre-filters and jumps directly if there is a single match.

Requires `fzf` on the client (`brew install fzf` on macOS, `apt install fzf` on Linux).

### Resulting directory structure

```
~/code/
├── github.com/
│   └── <ORG>/
│       ├── project-a/
│       └── project-b/
└── _inbox/                ← repos to reorganize (optional)
```

Same layout on client and server. ghq manages both identically.

---

## 8.3 Unversioned files in ~/code

These files live in the root `~/code/` directory and are not checked into any repo. They serve as local reference and as instructions for AI coding assistants.

### ~/code/README.md

A quick-reference file for navigating repos:

```bash
cat > ~/code/README.md << 'EOF'
# code/

Local repos managed with ghq (github.com/x-motemen/ghq).
Structure: mirrors the GitHub host/org/repo hierarchy.

## Navigation

    j                  jump to a repo (ghq list -p | fzf), requires fzf
    ghq list           list all repos
    ghq get <repo>     clone and place in the right location
    ghq list -p | fzf  explicit version of j

## Structure

    ~/code/
    └── github.com/
        └── <org>/
            ├── repo-a/
            └── repo-b/
EOF
```

### ~/code/.claude/CLAUDE.md

Claude Code reads this file automatically when launched from any subdirectory of `~/code/`. Use it to document repo conventions so the assistant follows them.

```bash
mkdir -p ~/code/.claude
cat > ~/code/.claude/CLAUDE.md << 'EOF'
# Development environment

Repos are managed with ghq. Structure: ~/code/github.com/<org>/<repo>.

## Creating a new repo

    mkdir -p ~/code/github.com/<ORG>/<NAME>
    cd ~/code/github.com/<ORG>/<NAME>
    git init
    gh repo create <ORG>/<NAME> --private
    git remote add origin git@github.com:<ORG>/<NAME>.git

## Cloning an existing repo

    ghq get github.com/<ORG>/<NAME>

## Conventions

- Commits: Conventional Commits (feat/fix/docs/chore)
- Default branch: main
- Credentials: never in plain text in versioned files, use <UPPERCASE_PLACEHOLDER>
EOF
```

---

## 8.4 Daily workflow

### Typical day

```
morning: k
  you are on the server in tmux session "main"

close laptop: Ctrl+A d  (detach)
  session keeps running on the server

resume later: k
  everything is exactly as you left it
```

The SSH connection can drop, the laptop can sleep, and the tmux session continues running. Reconnecting picks up where you left off.

### Essential tmux commands

| Key | Action |
|-----|--------|
| `Ctrl+A c` | new window |
| `Ctrl+A \|` | split vertically |
| `Ctrl+A -` | split horizontally |
| `Ctrl+A <number>` | switch to window N |
| `Ctrl+A d` | detach (session keeps running) |
| `Ctrl+A r` | reload tmux.conf |
| `Alt+arrows` | navigate between panes |
| `Ctrl+A [` | scroll mode (q to exit) |

### Session management

```bash
tmux ls                        # list sessions
tmux new-session -A -s main    # attach or create "main"
tmux new-session -s work       # new separate session
tmux switch -t work            # switch session
tmux kill-session -t work      # close session
```

> The prefix key (`Ctrl+A` with the config from [03-tmux.md](03-tmux.md)) is pressed and released before the next key, not held simultaneously: `Ctrl+A` then release, then `c`.

---

## 8.5 VS Code / Cursor Remote SSH

VS Code and Cursor can connect directly to the server over SSH, giving you a full IDE experience on remote files.

1. `Cmd+Shift+P` (or `Ctrl+Shift+P`) then `Remote-SSH: Connect to Host`
2. Select `<SERVER_ALIAS>` (must be defined in `~/.ssh/config`)
3. First connection installs the remote server component automatically (~30 seconds)
4. `File > Open Folder` and navigate to the project path (e.g., `/home/<USER>/code`)

File tree, diffs, and staging all reflect the remote filesystem in real time. Changes made in the editor are immediately visible to Claude Code running in tmux, and vice versa.

---

## 8.6 Mobile access

**Termius** (iOS/Android) supports SSH with key authentication and has native Tailscale support.

Configuration:
- Host: `<SERVER_IP>` (use the Tailscale IP if connecting over the mesh)
- User: `<USER>`
- SSH key: import your private key

Once connected:

```bash
tmux attach -t main
```

You get the same persistent session as from your laptop.

Terminal apps like iTerm2, Warp, or other SSH clients can also be configured with the same `ssh -t <SERVER_ALIAS> "tmux new-session -A -s main"` command, either as a saved profile or workflow.

---

## 8.7 Adding a new node

Checklist for setting up a new remote server with this workflow:

1. Run runbooks [01](01-shell.md) through [03](03-tmux.md) on the new server (shell, dev tools, tmux)
2. Generate an SSH key and add it to GitHub as both authentication and signing key (see [02-devtools.md](02-devtools.md))
3. Add a `Host <alias>` block in `~/.ssh/config` on your client machine
4. Add connection aliases (`k`, `ks`) in your client shell config, pointing to the new alias
5. Clone your repos with `ghq get` to have the same workspace immediately

---

## Notes

- autossh maintains network tunnels, tmux maintains processes: these are two independent layers. A broken tunnel does not interrupt the tmux session; just reconnect.
- For shared projects, consider adding `.devcontainer/devcontainer.json` to repos for reproducible environments across team members.
