---
name: boxed-marimo
description: >-
  Start a marimo notebook inside agent-owned/container-owned tmux and pair with
  the user via marimo-pair. Use for `*inabox` marimo pair programming workflows;
  depends on boxed-tmux and marimo-team/marimo-pair.
---

# boxed-marimo: marimo pair programming in boxed tmux

Use this skill to start a marimo server where the agent runs, keep it visible in
boxed-tmux, report the browser URL, then pair through marimo-pair after the user
opens the notebook.

Dependencies:

- `/boxed-tmux` skill for tmux socket/session management.
- `/marimo-pair` skill. If missing, ask the user for confirmation, then install
  it project-locally/non-interactively with
  `npx -y skills add marimo-team/marimo-pair --agent opencode --skill marimo-pair -y`.

## Workflow

1. Ensure `/marimo-pair` is installed; if missing, confirm and run
   `npx -y skills add marimo-team/marimo-pair --agent opencode --skill marimo-pair -y`
   in the project.
2. Use boxed-tmux bootstrap (`WORKDIR`, `SOCKET`, `SESSION`).
3. Start marimo in a persistent tmux window named `marimo-server`.
4. Capture the pane, extract/report the URL, and ask the user to open it.
5. Wait for the user to confirm the notebook is open.
6. Use marimo-pair `execute-code.sh --url <url>` and send a ready toast.

## Start marimo in boxed tmux

First use `/boxed-tmux` Constants/bootstrap to define `WORKDIR`, `SOCKET`, and
`SESSION`. Then start marimo. Default notebook name is `notebook_mo.py`; change
it if the user asks.

```bash
WINDOW=marimo-server
NOTEBOOK="${NOTEBOOK:-notebook_mo.py}"

tmux -S "$SOCKET" kill-window -t "$SESSION:$WINDOW" 2>/dev/null || true
tmux -S "$SOCKET" new-window -d -t "$SESSION:" -n "$WINDOW" -c "$WORKDIR" bash
tmux -S "$SOCKET" send-keys -t "$SESSION:$WINDOW" \
  "uvx marimo@latest edit $NOTEBOOK --sandbox --no-token --headless" C-m
sleep 4
tmux -S "$SOCKET" capture-pane -t "$SESSION:$WINDOW" -p -S -120
```

Tell the user the printed URL, usually `http://localhost:2718` or the next free
port. If no URL appears yet, wait briefly and capture again.

Always print the URL directly. If `boxed-dbus open-uri` is already available and
verified, use it to offer the host-side notification-action/OpenURI flow for the
marimo URL. If `boxed-dbus open-uri` is not available, do not bootstrap DBus just
for this; briefly mention that boxed-dbus can provide a clickable
notification/open action once the user starts the DBus proxy, then ask the user
to open the printed URL and confirm once the notebook is open.

## Connect after the user opens the URL

After the notebook is open, follow `/marimo-pair` for all notebook interaction;
use its `execute-code.sh --url <url>` and code_mode workflow.

Do not call `execute-code.sh` before the browser notebook is open; marimo needs
an active notebook session.

After the user confirms, send a toast:

```bash
bash "$WORKDIR/.agents/skills/marimo-pair/scripts/execute-code.sh" --url http://localhost:2718/ <<'PY'
import marimo as mo
mo.status.toast("📦🍃 boxed-marimo is connected — ready to pair!", kind="info")
PY
```

Use the actual URL/port reported by marimo.

## Notes

- Keep the tmux window alive while pairing.
- If the toast fails with “No active sessions”, ask the user to open/refresh the
  URL and try again.
- Edit live notebooks through marimo-pair/code_mode, not by writing the `.py`
  file directly.

## Metadata
* Author: thias <github.attic@typedef.net>, OpenAI gpt-5.5
* License: CC BY 4.0
* Version: 0.1
* Date: 2026-05-27
