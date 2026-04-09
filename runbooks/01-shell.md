# 01 - Shell

Install zsh with plugins, a modern prompt, and CLI tools that replace slow defaults. This is the foundation for everything else - later runbooks add lines to the shell profile created here.

> Why zsh first: later runbooks (dev tools, nvm, etc.) add config to the shell profile. If you install zsh after those, you have to manually migrate everything from `.bashrc` to `.zshrc`. Do this first to avoid that work.

## Placeholders

| Placeholder | Description | Example |
|---|---|---|
| `<USER>` | Your username on this machine | `deploy` |

## Step 1 - Install zsh and set it as default shell

Precondition: Ubuntu 24.04, sudo access.

```bash
sudo apt install -y zsh
sudo chsh -s /usr/bin/zsh <USER>
```

> Note: `chsh` without sudo prompts for a password interactively, which fails in non-tty SSH sessions. With sudo it skips the prompt. The path `/usr/bin/zsh` is correct on Ubuntu 24.04 - verify with `which zsh` before running.

Log out and log back in to activate the new shell.

Verify:

```bash
echo $SHELL
```

Expected: `/usr/bin/zsh`.

## Step 2 - Plugin manager: antidote

Antidote loads zsh plugins from a simple text file (one plugin per line). It clones them on first load and keeps them updated.

```bash
git clone --depth=1 https://github.com/mattmc3/antidote.git ~/.antidote
```

Verify:

```bash
ls ~/.antidote/antidote.zsh
```

## Step 3 - Prompt: starship

Cross-shell prompt written in Rust. Shows git branch, working tree status, language version, Docker context - only when relevant. Configured via a single TOML file, easy to sync across machines. Works without Nerd Fonts (icons are optional).

```bash
curl -sS https://starship.rs/install.sh | sudo sh -s -- --yes
```

> Note: the installer writes to `/usr/local/bin`, so it needs sudo.

Verify:

```bash
starship --version
```

## Step 4 - Modern CLI tools

All available via apt, all single binaries with no runtime dependencies. Removable with `apt remove` without residues.

```bash
sudo apt install -y zoxide fzf eza bat fd-find ripgrep btop git-delta
```

| Tool | Replaces | What it adds |
|------|----------|-------------|
| `zoxide` | `cd` | Learns visited paths and suggests them by partial name. `z proj` instead of `cd /home/user/projects`. |
| `fzf` | - | Interactive fuzzy finder. `Ctrl+R` for history search, `Ctrl+T` for file search. Has its own shell integration (do not load via antidote). |
| `eza` | `ls` | Colors, icons, git column (staged/modified), human-readable sizes. |
| `bat` | `cat` | Syntax highlighting, line numbers, automatic paging. On Ubuntu the binary is called `batcat`. |
| `fd-find` | `find` | Simple syntax, ignores `.git` and `.gitignore` automatically, fast. On Ubuntu the binary is called `fdfind`. |
| `ripgrep` | `grep` | Searches file contents, respects `.gitignore`, very fast on large repos. Used internally by VS Code and Claude Code. |
| `btop` | `htop` | Terminal resource monitor: CPU, RAM, disk, network, processes in one screen. |
| `git-delta` | git diff | `git diff` and `git log -p` with syntax highlighting and readable side-by-side diffs. |

Verify:

```bash
eza --version && batcat --version && fdfind --version && rg --version && zoxide --version && fzf --version && btop --version && delta --version
```

## Step 5 - Create `.zshrc`

The file is organized in blocks, top to bottom:

1. **History** - persistent history: saves to disk after every command (`INC_APPEND_HISTORY`), removes duplicates, ignores commands preceded by a space. Goes first because it must be active before any plugin loads.
2. **Plugin manager** - antidote loads plugins listed in `.zsh_plugins.txt` (step 6).
3. **Prompt and navigation** - starship (prompt), zoxide (`z` as smart cd), fzf (`Ctrl+R` history, `Ctrl+T` files).
4. **PATH and runtimes** - local paths, CUDA (if installed), nvm (if installed).
5. **Aliases** - replacements using the modern CLI tools from step 4.
6. **Functions** - `j()` for navigating repos with ghq + fzf (ghq is installed in [runbook 02](02-devtools.md)).
7. **Secrets** - loads credentials from a separate unversioned file (step 7).

```bash
cat > ~/.zshrc << 'EOF'
# history
HISTFILE=~/.zsh_history
HISTSIZE=100000
SAVEHIST=100000
setopt EXTENDED_HISTORY
setopt INC_APPEND_HISTORY
setopt HIST_IGNORE_ALL_DUPS
setopt HIST_IGNORE_SPACE
setopt HIST_FIND_NO_DUPS

# antidote
source ~/.antidote/antidote.zsh
antidote load ~/.zsh_plugins.txt

# starship prompt
eval "$(starship init zsh)"

# zoxide (smart navigation, learns frequently used paths)
eval "$(zoxide init zsh)"

# fzf shell integration (Ctrl+R history search, Ctrl+T file search)
source /usr/share/doc/fzf/examples/key-bindings.zsh
source /usr/share/doc/fzf/examples/completion.zsh

# PATH
export PATH="$PATH:$HOME/.local/bin"

# CUDA (if installed)
if [[ -d /usr/local/cuda ]]; then
  export PATH="$PATH:/usr/local/cuda/bin"
fi

# nvm (if installed)
export NVM_DIR="$HOME/.nvm"
[ -s "$NVM_DIR/nvm.sh" ] && \. "$NVM_DIR/nvm.sh"

# aliases - modern CLI tools
alias ls='eza --icons'
alias ll='eza -la --icons --git'
alias cat='batcat --paging=never'   # on Ubuntu the binary is batcat, not bat
alias find='fdfind'                 # on Ubuntu the binary is fdfind, not fd

# repo navigation (requires ghq from runbook 02)
# function instead of alias: accepts optional argument as pre-filled filter
j() {
  local dir
  dir=$(ghq list -p | fzf -q "${1:-}" --select-1 --exit-0)
  [[ -n "$dir" ]] && cd "$dir"
}

# local credentials (not versioned)
[[ -f ~/.zshrc.secrets ]] && source ~/.zshrc.secrets
EOF
```

### Zoxide usage

After visiting a few directories, zoxide learns them and lets you jump by partial name:

```bash
z proj       # jumps to the most frequently visited path containing "proj"
zi           # interactive selection with fzf
```

## Step 6 - Create `~/.zsh_plugins.txt`

This file lists the zsh plugins loaded by antidote (step 2) at shell startup. One plugin per line, `owner/repo` format from GitHub.

```bash
cat > ~/.zsh_plugins.txt << 'EOF'
# must have
zsh-users/zsh-autosuggestions
zdharma-continuum/fast-syntax-highlighting

# useful
zsh-users/zsh-completions
EOF
```

| Plugin | What it does |
|--------|-------------|
| `zsh-autosuggestions` | Suggests commands from history in gray as you type. Right arrow to accept. |
| `fast-syntax-highlighting` | Colors the command line: green if valid, red if the binary does not exist. Catch typos before pressing enter. |
| `zsh-completions` | Additional completions for tools not covered by base zsh (Docker, git, etc.). |

> Note: fzf does not belong in antidote. `junegunn/fzf` is a full application, not a zsh plugin. Install it via apt (step 4) and activate it through the integration files in `.zshrc` (step 5).

## Step 7 - Credentials: `.zshrc.secrets`

Tokens and credentials must not go in `.zshrc` (risk of ending up in a git repo or shared backup). Use a separate unversioned file:

```bash
touch ~/.zshrc.secrets
chmod 600 ~/.zshrc.secrets

# add your tokens, e.g.:
echo 'export HF_TOKEN="<YOUR_HUGGINGFACE_TOKEN>"' >> ~/.zshrc.secrets
echo 'export ANTHROPIC_API_KEY="<YOUR_ANTHROPIC_KEY>"' >> ~/.zshrc.secrets
```

The `.zshrc` already includes `[[ -f ~/.zshrc.secrets ]] && source ~/.zshrc.secrets` at the end. The line does not error if the file does not exist yet.

## Step 8 - Verify

```bash
exec zsh   # reload shell without logout

echo $SHELL          # /usr/bin/zsh
starship --version
eza --version
batcat --version
fdfind --version
rg --version
```

Expected: `/usr/bin/zsh` and version strings for each tool.

---

## Appendix - Migration from bash

If the machine already had bash in use and you switch to zsh later, `.bashrc` contains lines added by previous installations that will no longer be read.

Show customizations in `.bashrc` (excludes Ubuntu boilerplate):

```bash
grep -v '^#\|^$\|case \$-\|HISTCONTROL\|histappend\|HISTSIZE\|checkwinsize\|lesspipe\|debian_chroot\|color_prompt\|PS1\|xterm\|dircolors\|alias ls\|alias ll\|alias la\|alias l=\|alert\|bash_aliases\|bash_completion\|shopt' ~/.bashrc
```

What to migrate to `.zshrc` (add after the PATH section):

| What to look for in `.bashrc` | Action |
|-------------------------------|--------|
| `export PATH=...` with custom paths | Migrate to `.zshrc` |
| `export VARIABLE=value` (non-sensitive) | Migrate to `.zshrc` |
| `export TOKEN=...` or credentials | Migrate to `.zshrc.secrets` |
| `eval "$(zoxide init bash)"` | Already in `.zshrc` as `zoxide init zsh`, do not copy |
| `eval "$(starship init bash)"` | Already in `.zshrc` as `starship init zsh`, do not copy |
| `export NVM_DIR` + nvm source | Already in `.zshrc`, do not copy |
| Ubuntu boilerplate (PS1, HISTCONTROL, shopt, alias ls/ll/grep) | Do not migrate: replaced by starship and eza aliases |

After migration, `.bashrc` stays intact (bash is still the shell for other users and system scripts). Do not delete it.
