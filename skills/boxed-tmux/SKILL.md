---
name: boxed-tmux
description: >-
  Let the agent run and control a tmux server using a project-local socket.
  Use tmux for long-running processes, dev servers, watchers, optional user
  attach/debug workflows, and tmux-based Pi subagents when work is
  longer/asynchronous or should preserve the main agent's context.
---

# boxed-tmux: agent-owned tmux workflow

Commands run where the tmux server runs. With the shared project socket, the
first agent to start tmux becomes the tmux host; later agents connect to that
server, so their tmux workloads run in the host agent's environment. If that
server dies, another agent can bootstrap a new one.

Use for long-running or interactive commands that should not block the agent:
dev servers, watchers, notebooks, debugging shells, and tmux-based Pi subagents.
The socket is project-local so the human can attach only when needed.

## Constants / bootstrap

Always use uppercase `-S`, one project-local socket, and one tmux session per
agent. If the user asks for a specific name, set `AGENT_ID` or `SESSION`;
otherwise derive a stable id from the tool-call parent process and its start time.

```bash
WORKDIR="${PWD:-/stage}"
SOCKET="$WORKDIR/.agents/run/tmux-socket"
PARENT_PID="${PPID:-$$}"
PARENT_START="$(awk '{print $22}' "/proc/$PARENT_PID/stat" 2>/dev/null || true)"
# AGENT_ID identifies the main agent, should be stable across calls
AGENT_ID="${AGENT_ID:-p${PARENT_PID}-${PARENT_START:-unknown}}"
AGENT_ID="$(printf '%s' "$AGENT_ID" | tr -cd '[:alnum:]_.-')"
SESSION="${SESSION:-agent-${AGENT_ID:-unknown}}"
mkdir -p "$WORKDIR/.agents/run"
if ! tmux -S "$SOCKET" has-session -t "$SESSION" 2>/dev/null; then
  tmux -S "$SOCKET" new-session -d -s "$SESSION" -c "$WORKDIR"
fi
```

Verify with:

```bash
tmux -S "$SOCKET" list-sessions
tmux -S "$SOCKET" list-windows -t "$SESSION"
```

## Long-running command pattern

Create a persistent shell window, then send the command. This keeps output
inspectable even if the command exits.

```bash
WINDOW=dev-server
COMMAND='npm run dev'

tmux -S "$SOCKET" kill-window -t "$SESSION:$WINDOW" 2>/dev/null || true
tmux -S "$SOCKET" new-window -d -t "$SESSION:" -n "$WINDOW" -c "$WORKDIR" bash
tmux -S "$SOCKET" send-keys -t "$SESSION:$WINDOW" "$COMMAND" C-m
sleep 2
tmux -S "$SOCKET" capture-pane -t "$SESSION:$WINDOW" -p -S -80
```

Prefer this over `nohup ... &` when visibility or interaction may matter.

## Pi subagents

Use tmux subagents only when useful: longer/asynchronous work, visible parallel
work, or preserving the main agent's context. For small direct questions, do the
work normally.

**Batch subagent**: fully specified task, no follow-up expected. Run `pi -p` in a
tmux window, require result/done files, and kill the window after reading.

```bash
# SUBAGENT_IDs should be unique per spawn
SUBAGENT_ID="${SUBAGENT_ID:-$(date +%s%N)}"
SUBAGENT_ID="$(printf '%s' "$SUBAGENT_ID" | tr -cd '[:alnum:]_.-')"
WINDOW="${WINDOW:-pi-batch-$SUBAGENT_ID}"
TASK_DIR="$WORKDIR/.agents/run/subagents/$SUBAGENT_ID"
mkdir -p "$TASK_DIR"
PROMPT="Do this task: ...

When finished, write your final answer to $TASK_DIR/result.md, then touch $TASK_DIR/done.
Do not touch done before result.md is complete."

tmux -S "$SOCKET" new-window -d -t "$SESSION:" -n "$WINDOW" -c "$WORKDIR" bash
tmux -S "$SOCKET" send-keys -t "$SESSION:$WINDOW" "pi -p $(printf '%q' \"$PROMPT\")" C-m

# Later:
if [ -f "$TASK_DIR/done" ]; then
  cat "$TASK_DIR/result.md"
  tmux -S "$SOCKET" kill-window -t "$SESSION:$WINDOW"
fi
```

**Interactive subagent**: use when clarifications/follow-ups or reusable context
are expected. Keep the window until no longer needed.

```bash
# SUBAGENT_IDs should be unique per spawn
SUBAGENT_ID="${SUBAGENT_ID:-$(date +%s%N)}"
SUBAGENT_ID="$(printf '%s' "$SUBAGENT_ID" | tr -cd '[:alnum:]_.-')"
WINDOW="${WINDOW:-pi-subagent-$SUBAGENT_ID}"
TASK_DIR="$WORKDIR/.agents/run/subagents/$SUBAGENT_ID"
mkdir -p "$TASK_DIR"

tmux -S "$SOCKET" new-window -d -t "$SESSION:" -n "$WINDOW" -c "$WORKDIR" bash
tmux -S "$SOCKET" send-keys -t "$SESSION:$WINDOW" 'pi' C-m
sleep 2

PROMPT="Do this task: ...

When finished, write your final answer to $TASK_DIR/result.md, then touch $TASK_DIR/done.
Do not touch done before result.md is complete."

tmux -S "$SOCKET" send-keys -l -t "$SESSION:$WINDOW" "$PROMPT"
sleep 0.5
tmux -S "$SOCKET" send-keys -t "$SESSION:$WINDOW" Enter
tmux -S "$SOCKET" capture-pane -t "$SESSION:$WINDOW" -p -S -80
```

Do not infer completion from the tmux UI unless this file/sentinel protocol was
not used.

## Common operations

```bash
# Capture output
tmux -S "$SOCKET" capture-pane -t "$SESSION:WINDOW" -p -S -120

# Interrupt
tmux -S "$SOCKET" send-keys -t "$SESSION:WINDOW" C-c

# Send literal text, then Enter
tmux -S "$SOCKET" send-keys -l -t "$SESSION:WINDOW" 'literal text'
tmux -S "$SOCKET" send-keys -t "$SESSION:WINDOW" C-m

# Kill a window
tmux -S "$SOCKET" kill-window -t "$SESSION:WINDOW"
```

## Human attach/debug

Optional, from the host project root if the socket is mounted and tmux versions
are compatible:

```bash
tmux -S .agents/run/tmux-socket list-sessions
tmux -S .agents/run/tmux-socket attach -t agent-<id>
```

## Troubleshooting

- `no server running`: run the bootstrap snippet.
- `permission denied`: check `.agents/run` and socket permissions.
- Window disappears: start a persistent `bash` window first, then `send-keys`.
- Host attach fails: check socket visibility and host/container tmux compatibility.

## Metadata
* Author: thias <github.attic@typedef.net>, OpenAI gpt-5.5
* License: CC BY 4.0
* Version: 0.1
* Date: 2026-05-27
