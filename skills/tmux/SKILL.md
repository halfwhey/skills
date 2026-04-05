---
name: tmux
description: Control tmux sessions, windows, and panes via the tmux CLI. Covers session management, window/pane operations, sending keys, reading output, and pane decoration.
version: 1.1.0
license: MIT
---

# tmux

Use the `tmux` CLI to interact with running tmux sessions.

## Golden Rule: Always Resolve pane_id First

Before interacting with any pane, resolve its unique `pane_id`. Use this as the canonical handle for all subsequent operations — it is stable, unambiguous, and survives window/session renames.

```bash
# From a user-supplied number (e.g. "pane 1" in the current window)
PANE_ID=$(tmux display-message -t :.1 -p '#{pane_id}')   # → %42

# From a full target
PANE_ID=$(tmux display-message -t 0:2.1 -p '#{pane_id}')

# pane_id is valid as a -t target for all pane-level commands
tmux send-keys -t %42 "echo hello" Enter
tmux capture-pane -t %42 -p
tmux select-pane -t %42 -T "my title"
tmux kill-pane -t %42
```

A pane's `pane_id` changes when it is destroyed and recreated — even at the same index. This means using `pane_id` automatically treats recreated panes as new sessions.

---

## Resolving Panes by Description

The user may refer to panes informally — "the other pane", "the pane running vim", "the shell on the right". Resolve these yourself before falling back to asking:

```bash
# List all panes in the current window (excluding yourself)
tmux list-panes -F '#{pane_id} #{pane_current_command}' | grep -v "^$TMUX_PANE "

# "The other pane" / "the only other pane" — works when there are exactly 2 panes
PANE_ID=$(tmux list-panes -F '#{pane_id}' | grep -v "^$TMUX_PANE$")

# "The pane running vim"
PANE_ID=$(tmux list-panes -F '#{pane_id} #{pane_current_command}' | grep vim | awk '{print $1}')
```

Use `$TMUX_PANE` (your own pane ID, set automatically by tmux) to exclude yourself from the list.

If the description is ambiguous or matches multiple panes, ask the user:

1. Run `tmux display-panes -d 5000` — numbered overlays appear for 5 seconds
2. Ask the user which number
3. Resolve with `:.N`:
   ```bash
   PANE_ID=$(tmux display-message -t :.1 -p '#{pane_id}')
   ```

---

## Reading Pane Output

Use the `bin/read-tmux` script — do **not** use raw `capture-pane` for reading command output.

```bash
bin/read-tmux %42          # delta mode (default) — for shell output
bin/read-tmux --tui %42    # TUI mode — diff of changed screen regions
bin/read-tmux --full %42   # TUI mode — always full screen (no diff)
```

### Delta mode (default)

- **First call**: captures full scrollback, saves position marker, prints ≤4KB
- **Subsequent calls**: captures only new lines since last read
- **No changes**: prints `(no new output)`

### TUI mode (`--tui`)

Use for any full-screen application (vim, htop, lazygit, top, etc.).

- **First call**: captures full visible screen, saves snapshot
- **Subsequent calls**: diffs against previous capture
  - If ≤50% of lines changed → shows unified diff (only changed regions)
  - If >50% of lines changed → shows full screen (heavy redraw fallback)
- **No changes**: prints `(no changes on screen)`

### Output capping

All output is capped at first 2000 + last 2000 bytes; middle is truncated if exceeded.

### State directory

State is stored in `/tmp/tmux-skill/<pane_id>/`:
```
position     ← line counter for delta mode
tui_prev     ← last TUI capture for diffing
```

---

## Pane Decoration

Both `bin/read-tmux` and `bin/send-tmux` auto-decorate panes on first interaction — a yellow "● Operated by Claude" bar appears at the top of the pane. No manual decoration needed.

To release decoration when done with a pane:
```bash
tmux set-option -p -t %42 -u pane-border-status 2>/dev/null || true
tmux set-option -p -t %42 -u pane-border-format 2>/dev/null || true
```

---

## Sending Commands

Use `bin/send-tmux` — it auto-decorates the pane and passes all arguments to `tmux send-keys`.

```bash
# Run a command
bin/send-tmux %42 "npm test" Enter

# Interrupt
bin/send-tmux %42 C-c

# Send EOF / exit shell
bin/send-tmux %42 C-d

# Quit a pager
bin/send-tmux %42 q
```

---

## Sessions

```bash
# List
tmux list-sessions
tmux list-sessions -F '#{session_name} (#{session_windows} windows)'

# Create (detached)
tmux new-session -d -s <name> [-c <start-dir>]

# Rename
tmux rename-session -t <old> <new>

# Kill
tmux kill-session -t <name>

# Check existence
tmux has-session -t <name> 2>/dev/null && echo exists
```

---

## Windows

```bash
# List
tmux list-windows -t <session>
tmux list-windows -t <session> -F '#{window_index}: #{window_name}'

# Create
tmux new-window -t <session> -n <name> [-c <dir>]

# Rename
tmux rename-window -t <session>:<index> <new-name>

# Move to another session or index
tmux move-window -s <session>:<index> -t <dst-session>:<new-index>

# Swap two windows
tmux swap-window -s <session>:<a> -t <session>:<b>

# Kill
tmux kill-window -t <session>:<index>
```

---

## Panes

```bash
# List (resolve pane_id for any pane you'll interact with)
tmux list-panes -t <session>:<window> -F '#{pane_index} #{pane_id} #{pane_current_command}'
tmux list-panes -a -F '#{session_name}:#{window_index}.#{pane_index} #{pane_id} #{pane_current_command}'

# Create — split relative to Claude's own pane so it opens in the right window.
# $TMUX_PANE is Claude's pane_id, set automatically by tmux.
# -P -F prints the new pane's id directly.
PANE_ID=$(tmux split-window -h -t "$TMUX_PANE" -P -F '#{pane_id}')   # side by side
PANE_ID=$(tmux split-window -v -t "$TMUX_PANE" -P -F '#{pane_id}')   # top/bottom

# Rename (sets title shown in pane border)
tmux select-pane -t %42 -T <title>

# Move pane to another window
tmux move-pane -s %42 -t <dst-session>:<dst-window>

# Break pane out into its own window
tmux break-pane -t %42 -n <new-window-name>

# Join a window into another as a pane
tmux join-pane -s <src-window> -t <dst-window>

# Swap two panes
tmux swap-pane -s %42 -t %99

# Kill
tmux kill-pane -t %42
```

---

## Querying State

```bash
# Pane properties
tmux display-message -t %42 -p '#{pane_id}'
tmux display-message -t %42 -p '#{pane_current_command}'
tmux display-message -t %42 -p '#{pane_current_path}'
tmux display-message -t %42 -p '#{pane_pid}'

# Useful format variables
# #{pane_id}              unique pane identifier (%42)
# #{pane_index}           position within window (can be reused after kill)
# #{pane_current_command} process running in pane
# #{pane_current_path}    working directory
# #{pane_pid}             PID of pane shell
# #{session_name}         session name
# #{window_index}         window number
# #{window_name}          window name
```

---

## Practical Patterns

### Poll until a command finishes

```bash
tmux send-keys -t %42 "make build && echo __DONE__" Enter
while ! bin/read-tmux %42 | grep -q "__DONE__"; do sleep 0.5; done
```

### Open a new pane for a project task

```bash
WINDOW=$(tmux new-window -t myproject -n build -P -F '#{window_index}')
PANE_ID=$(tmux display-message -t "myproject:${WINDOW}.0" -p '#{pane_id}')
tmux select-pane -t "$PANE_ID" -T "● claude"
tmux send-keys -t "$PANE_ID" "make build" Enter
```

### Interrupt and re-run

```bash
tmux send-keys -t %42 C-c
sleep 0.2
tmux send-keys -t %42 "make build" Enter
```
