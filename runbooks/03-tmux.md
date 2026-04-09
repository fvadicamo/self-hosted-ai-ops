# 03 - tmux

Terminal multiplexer for persistent sessions on remote servers. Sessions survive SSH disconnections and allow splitting the screen into multiple panes. Essential for headless remote work.

This runbook installs tmux and applies an opinionated configuration that changes several defaults for comfort and usability.

## Configuration choices

| Change | Default | This config | Why |
|--------|---------|-------------|-----|
| Prefix key | `Ctrl+B` | `Ctrl+A` | More comfortable reach on most keyboards |
| Mouse support | off | on | Scroll output, click to select panes |
| Split bindings | `"` and `%` | `\|` and `-` | Mnemonic: vertical bar, horizontal dash |
| Pane navigation | prefix + arrow | `Alt+arrow` | No prefix needed, faster switching |
| Scrollback | 2000 lines | 10000 lines | Default is insufficient for long command output |
| Window numbering | starts at 0 | starts at 1 | Matches keyboard layout (1 is leftmost) |

## Placeholders

None. This runbook uses no placeholders.

---

## Install tmux

**Precondition:** Ubuntu 24.04 with apt available.

**Action:**

```bash
sudo apt install -y tmux
```

**Verify:**

```bash
tmux -V
```

Expected output: `tmux 3.4` or newer.

---

## Write configuration

**Precondition:** tmux is installed. No existing `~/.tmux.conf` (or you are fine overwriting it).

**Action:**

```bash
cat > ~/.tmux.conf << 'EOF'
# Prefix: Ctrl+A
unbind C-b
set -g prefix C-a
bind C-a send-prefix

# Mouse
set -g mouse on

# Window index from 1
set -g base-index 1
set -g pane-base-index 1
set -g renumber-windows on

# Mnemonic splits
bind | split-window -h -c '#{pane_current_path}'
bind - split-window -v -c '#{pane_current_path}'

# Navigate panes with Alt+arrows
bind -n M-Left select-pane -L
bind -n M-Right select-pane -R
bind -n M-Up select-pane -U
bind -n M-Down select-pane -D

# Scrollback
set -g history-limit 10000

# Minimal status bar
set -g status-style 'bg=colour235 fg=colour136'
set -g status-left '[#S] '
set -g status-right '%H:%M %d-%b'
set -g status-right-length 30

# Terminal colors
set -g default-terminal 'screen-256color'
set -ga terminal-overrides ',xterm-256color:Tc'

# Reload config
bind r source-file ~/.tmux.conf \; display 'config reloaded'
EOF
```

**Verify:**

```bash
grep 'prefix C-a' ~/.tmux.conf
```

Expected output: `set -g prefix C-a`

---

## Verify tmux starts with the new config

**Precondition:** `~/.tmux.conf` exists with the configuration above.

**Action:**

```bash
tmux new-session -d -s test
tmux show-option -g prefix -t test
tmux kill-session -t test
```

**Verify:**

The `show-option` command should output:

```
prefix C-a
```

---

## Quick reference

> Note: the prefix key is `Ctrl+A`, not the default `Ctrl+B`. On Mac, use the Ctrl key (not Cmd). The letter after the prefix is pressed in sequence, not simultaneously: press `Ctrl+A`, release, then press the next key.

### Session management

| Action | Command |
|--------|---------|
| New named session | `tmux new -s name` |
| List sessions | `tmux ls` |
| Attach to session | `tmux attach -t name` |
| Detach | `Ctrl+A` then `d` |
| Kill session | `tmux kill-session -t name` |

### Windows (tabs)

| Action | Keys |
|--------|------|
| New window | `Ctrl+A` then `c` |
| Next window | `Ctrl+A` then `n` |
| Previous window | `Ctrl+A` then `p` |
| Select window by number | `Ctrl+A` then `0-9` |
| Rename window | `Ctrl+A` then `,` |

### Panes (splits)

| Action | Keys |
|--------|------|
| Vertical split | `Ctrl+A` then `\|` |
| Horizontal split | `Ctrl+A` then `-` |
| Navigate panes | `Alt+arrow` (no prefix needed) |
| Close pane | `exit` or `Ctrl+D` |
| Toggle zoom (fullscreen pane) | `Ctrl+A` then `z` |

### Other

| Action | Keys |
|--------|------|
| Scroll mode (copy mode) | `Ctrl+A` then `[` |
| Exit scroll mode | `q` |
| Reload config | `Ctrl+A` then `r` |
