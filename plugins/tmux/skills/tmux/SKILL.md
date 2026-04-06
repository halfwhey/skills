---
name: tmux
description: Use this skill whenever you need to interact with tmux — sending commands to panes, reading their output, watching a TUI app like vim, htop, or lazygit, or managing sessions and windows. Reach for it any time the user says "run this in the terminal", "check that pane", "send a command to the shell", or when you spawn a background task that you'll need to monitor. Always use this skill when tmux is involved — it provides reliable pane identification, delta output reading, and safe key-sending that prevent common mistakes.
---

# tmux

Interact with tmux sessions using the `tmux` CLI and the bundled `read-tmux` and `send-tmux` helper scripts. The scripts live at `{baseDir}/scripts/` — substitute the skill's base directory when invoking them.

## Resolve pane_id first

Before doing anything with a pane, resolve its `pane_id`. This stable, unambiguous handle survives window/session renames — and since a recreated pane gets a new `pane_id`, using it automatically treats recreated panes as new sessions.

```bash
# From a user-supplied index (e.g. "pane 1" in the current window)
PANE_ID=$(tmux display-message -t :.1 -p '#{pane_id}')   # → %42

# From a full target
PANE_ID=$(tmux display-message -t 0:2.1 -p '#{pane_id}')

# pane_id works as -t for all pane-level commands
tmux send-keys -t %42 "echo hello" Enter
tmux capture-pane -t %42 -p
tmux select-pane -t %42 -T "my title"
tmux kill-pane -t %42
```

## Finding a pane from an informal description

The user may say "the other pane", "the pane running vim", "the shell on the right". Resolve these yourself before asking:

```bash
# All panes in current window, excluding your own
tmux list-panes -F '#{pane_id} #{pane_current_command}' | grep -v "^$TMUX_PANE "

# "The other pane" — works when there are exactly 2 panes
PANE_ID=$(tmux list-panes -F '#{pane_id}' | grep -v "^$TMUX_PANE$")

# "The pane running vim"
PANE_ID=$(tmux list-panes -F '#{pane_id} #{pane_current_command}' | grep vim | awk '{print $1}')
```

`$TMUX_PANE` is set automatically by tmux — it's your own pane ID.

If the description is still ambiguous, show numbered overlays:

```bash
tmux display-panes -d 5000   # numbered overlays appear for 5 seconds
# ask the user which number, then resolve
PANE_ID=$(tmux display-message -t :.N -p '#{pane_id}')
```

---

## Reading output

Use `read-tmux`, not raw `tmux capture-pane`. The script tracks position between calls so you only receive new output, and it handles TUI apps separately from shell panes.

```bash
{baseDir}/scripts/read-tmux %42          # shell panes — delta since last read
{baseDir}/scripts/read-tmux --tui %42    # TUI apps (vim, htop, etc.) — diff of changed screen regions
{baseDir}/scripts/read-tmux --full %42   # TUI — always show full screen, no diff
```

**Delta mode** (default):
- First call: captures full scrollback up to cursor, saves position
- Subsequent calls: only new lines since last read
- No new output: prints `(no new output)`

**TUI mode** (`--tui`):
- First call: captures full visible screen, saves snapshot
- Subsequent calls: unified diff if ≤50% of lines changed; full screen otherwise
- No changes: prints `(no changes on screen)`

Output is capped at first 2000 + last 2000 bytes. State lives in `/tmp/tmux-skill/<pane_id>/`.

---

## Sending keys

Use `send-tmux` — it decorates the pane and forwards all arguments to `tmux send-keys`.

```bash
{baseDir}/scripts/send-tmux %42 "npm test" Enter   # run a command
{baseDir}/scripts/send-tmux %42 C-c                # interrupt
{baseDir}/scripts/send-tmux %42 C-d                # EOF / exit shell
{baseDir}/scripts/send-tmux %42 q                  # quit a pager
```

---

## Pane decoration

Both `read-tmux` and `send-tmux` auto-decorate panes on first use — a yellow "● Operated by Claude" bar appears at the top. No manual setup needed.

To release when done:

```bash
tmux set-option -p -t %42 -u pane-border-status 2>/dev/null || true
tmux set-option -p -t %42 -u pane-border-format 2>/dev/null || true
```

---

## Sessions

```bash
tmux list-sessions
tmux list-sessions -F '#{session_name} (#{session_windows} windows)'
tmux new-session -d -s <name> [-c <start-dir>]
tmux rename-session -t <old> <new>
tmux kill-session -t <name>
tmux has-session -t <name> 2>/dev/null && echo exists
```

## Windows

```bash
tmux list-windows -t <session>
tmux list-windows -t <session> -F '#{window_index}: #{window_name}'
tmux new-window -t <session> -n <name> [-c <dir>]
tmux rename-window -t <session>:<index> <new-name>
tmux move-window -s <session>:<index> -t <dst-session>:<new-index>
tmux swap-window -s <session>:<a> -t <session>:<b>
tmux kill-window -t <session>:<index>
```

## Panes

```bash
# List
tmux list-panes -t <session>:<window> -F '#{pane_index} #{pane_id} #{pane_current_command}'
tmux list-panes -a -F '#{session_name}:#{window_index}.#{pane_index} #{pane_id} #{pane_current_command}'

# Create — split relative to your own pane so it opens in the right window
PANE_ID=$(tmux split-window -h -t "$TMUX_PANE" -P -F '#{pane_id}')   # side by side
PANE_ID=$(tmux split-window -v -t "$TMUX_PANE" -P -F '#{pane_id}')   # top/bottom

tmux select-pane -t %42 -T <title>
tmux move-pane -s %42 -t <dst-session>:<dst-window>
tmux break-pane -t %42 -n <new-window-name>
tmux join-pane -s <src-window> -t <dst-window>
tmux swap-pane -s %42 -t %99
tmux kill-pane -t %42
```

## Querying state

```bash
tmux display-message -t %42 -p '#{pane_id}'
tmux display-message -t %42 -p '#{pane_current_command}'
tmux display-message -t %42 -p '#{pane_current_path}'
tmux display-message -t %42 -p '#{pane_pid}'
```

Useful format variables: `#{pane_id}`, `#{pane_index}`, `#{pane_current_command}`, `#{pane_current_path}`, `#{pane_pid}`, `#{session_name}`, `#{window_index}`, `#{window_name}`

---

## Patterns

### Wait for a command to finish

```bash
{baseDir}/scripts/send-tmux %42 "make build && echo __DONE__" Enter
while ! {baseDir}/scripts/read-tmux %42 | grep -q "__DONE__"; do sleep 0.5; done
```

### Open a new pane for a task

```bash
WINDOW=$(tmux new-window -t myproject -n build -P -F '#{window_index}')
PANE_ID=$(tmux display-message -t "myproject:${WINDOW}.0" -p '#{pane_id}')
{baseDir}/scripts/send-tmux "$PANE_ID" "make build" Enter
```

### Interrupt and re-run

```bash
{baseDir}/scripts/send-tmux %42 C-c
sleep 0.2
{baseDir}/scripts/send-tmux %42 "make build" Enter
```
