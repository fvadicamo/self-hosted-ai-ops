# 03 - tmux

Terminal multiplexer for persistent sessions on remote servers. Sessions survive SSH disconnections and allow splitting the screen into multiple panes. Essential for headless remote work.

This runbook installs tmux and applies an opinionated configuration that changes several defaults for comfort and usability, and enables capabilities (true color, system clipboard via OSC 52, extended keys, OSC/DCS passthrough, focus events) that modern interactive CLIs and TUIs rely on.

## Configuration choices

| Change | Default | This config | Why |
|--------|---------|-------------|-----|
| Prefix key | `Ctrl+B` | `Ctrl+A` | More comfortable reach on most keyboards |
| Mouse support | off | on | Scroll output, click to select panes |
| Split bindings | `"` and `%` | `\|` and `-` | Mnemonic: vertical bar, horizontal dash |
| Pane navigation | prefix + arrow | `Alt+arrow` | No prefix needed, faster switching |
| Scrollback | 2000 lines | 10000 lines | Default is insufficient for long command output |
| Window numbering | starts at 0 | starts at 1 | Matches keyboard layout (1 is leftmost) |
| Default terminal | `screen` | `tmux-256color` | Modern terminfo with extended capabilities (requires `ncurses-term`) |
| Terminal features | (auto-detected) | `RGB hyperlinks usstyle sync clipboard extkeys osc7` + `focus` | Advertise outer-terminal support so tmux forwards true color, OSC 8 hyperlinks, underline styles, synchronized output, OSC 52 clipboard, modified keys via CSI u, OSC 7 cwd reporting, and focus events |
| OSC/DCS passthrough | off | on | Allow inner programs to emit escape sequences directly to the host terminal (required by interactive CLIs and some TUIs that paint UI via OSC/DCS) |
| Focus events | off | on | Forward terminal focus-in/focus-out to inner programs (autosave, notifications, etc.) |
| Extended keys | off | on (server option) | Report modified keys (Shift+Enter, Ctrl+Shift+letter, ...) via CSI u so inner programs can distinguish them |

> Note on the outer terminal: capabilities advertised here only work end-to-end if the terminal emulator running on the client (iTerm2, Alacritty, WezTerm, kitty, GNOME Terminal, etc.) actually supports them and is configured to do so (true color enabled, clipboard access from terminal applications allowed, modifier reporting enabled). Check your terminal's documentation if a feature does not seem to work.

## Placeholders

None. This runbook uses no placeholders.

---

## Install tmux

**Precondition:** Ubuntu 24.04 with apt available.

**Action:**

```bash
sudo apt install -y tmux ncurses-term
```

`ncurses-term` ships the `tmux-256color` terminfo entry used below.

**Verify:**

```bash
tmux -V
infocmp tmux-256color | head -1
```

Expected: `tmux 3.4` or newer, and the terminfo entry exists (no error).

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

# Terminal capabilities (true color, hyperlinks, system clipboard via OSC 52,
# synchronized output, extended keys via CSI u, OSC 7 cwd reporting)
set -g default-terminal 'tmux-256color'
set -as terminal-features 'xterm*:RGB:hyperlinks:usstyle:sync:clipboard:extkeys:osc7'
set -as terminal-features '*:focus'

# Passthrough OSC/DCS sequences from inner programs to the outer terminal
# (required by interactive CLIs and TUIs that paint UI via escape sequences)
set -g allow-passthrough on

# Forward terminal focus-in/focus-out events to inner programs
set -g focus-events on

# Report modified keys (Shift+Enter, Ctrl+Shift+letter, ...) via CSI u
set -s extended-keys on

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
tmux show-option -g allow-passthrough -t test
tmux show-option -s extended-keys -t test
tmux kill-session -t test
```

**Verify:**

The output should show:

```
prefix C-a
allow-passthrough on
extended-keys on
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
