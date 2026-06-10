---
name: boxed-dbus
description: >-
  Use a narrow, host-created xdg-dbus-proxy socket from a containerized agent.
  Use for desktop notifications and other explicitly proxied session-bus
  interfaces without exposing the full host DBus session bus.
---

# boxed-dbus: filtered host DBus proxy workflow

Use this skill when the agent needs limited access to the host desktop session
DBus from inside a boxed/containerized workflow. The default supported use case
is sending desktop notifications through `org.freedesktop.Notifications`.

Do **not** ask the user to expose the full `DBUS_SESSION_BUS_ADDRESS` unless they
explicitly want that. Prefer a filtered `xdg-dbus-proxy` socket.

This workflow assumes a Linux desktop host with a freedesktop-compatible session
DBus and notification service. If the proxy cannot be created because the host is
Windows/macOS/WSL without a compatible DBus bridge, explain that this skill is
not applicable instead of asking for the full host bus.

## Calling convention

Use these profile names in prompts and status messages:

- `notifications`: default profile; only `org.freedesktop.Notifications`.
- `open-uri`: optional profile; adds `org.freedesktop.portal.Desktop` for
  `org.freedesktop.portal.OpenURI.OpenURI`.

Default to `notifications` unless the user explicitly asks to open a host URL or
use the desktop portal.

## Capability profiles

Use the narrowest profile that satisfies the task.

**Default profile: notifications**

- Allows desktop notifications through `org.freedesktop.Notifications`.
- Use for progress updates, completion/failure alerts, and notification actions.
- This is the default and should be preferred unless the user asks for more.

**Optional profile: Desktop Portal / OpenURI**

- Extends the default profile with `org.freedesktop.portal.Desktop`.
- Use only when the user wants host desktop portal integration, especially
  opening URLs through `org.freedesktop.portal.OpenURI.OpenURI`.
- Broader than notifications-only: the same portal service exposes interfaces
  such as Screenshot, ScreenCast, RemoteDesktop, Camera, Location, FileChooser,
  and others. Portal backends may prompt, and grants may persist.
- Do not enable this profile silently. Explain the broader capability and ask the
  user to restart the host proxy with the optional profile command.

## Project-local DBus proxy socket

Use one project-local proxy socket by convention:

```bash
WORKDIR="${PWD:-/stage}"
PROXY_SOCKET="$WORKDIR/.agents/run/dbus-host-proxy"
```

Use this path in prompts, checks, and examples unless the user explicitly asks
for a different socket path.

## User-side bootstrap: default notifications profile

The default proxy must be started by the human on the host, from the project
root, so it can access the host desktop session bus:

```bash
mkdir -p .agents/run
PROXY_SOCKET="$PWD/.agents/run/dbus-host-proxy"
rm -f "$PROXY_SOCKET"
xdg-dbus-proxy "${DBUS_SESSION_BUS_ADDRESS}" "$PROXY_SOCKET" \
  --filter \
  --talk=org.freedesktop.Notifications
```

Keep that process running. The agent can then use:

```bash
export DBUS_SESSION_BUS_ADDRESS="unix:path=$PWD/.agents/run/dbus-host-proxy"
```

This grants access only to the interfaces/names allowed by the proxy rule. With
the default command, expect notification access but not portal or general session
bus access. If a task needs the optional Desktop Portal profile, ask the user to
restart the proxy with the optional profile command below.

## Agent check / bootstrap prompt

Before using the proxy, verify that the socket exists and responds. Prefer
`gdbus` for all checks and calls. If `gdbus` is missing, install it on demand
inside the container; on Debian/Ubuntu it is provided by `libglib2.0-bin`:

```bash
apt-get update && apt-get install -y libglib2.0-bin
```

If the proxy socket is missing or inactive, instruct the user to run the
user-side bootstrap above, then stop and wait for the user to say it is running.
When the user says it is running, run the check again. Do not wait inside the
tool call for the human to start it.

Use the following snippets as small helpers, not as one monolithic script.
Define the project-local socket path before using the helpers:

```bash
WORKDIR="${PWD:-/stage}"
PROXY_SOCKET="$WORKDIR/.agents/run/dbus-host-proxy"
```

If the socket is missing, print the default host bootstrap command and stop:

```bash
prompt_start_proxy() {
  printf 'No active host DBus proxy socket found at:\n  %s\n\n' "$PROXY_SOCKET"
  cat <<'MSG'
Please run the default host bootstrap command from the project root and
keep it running:

mkdir -p .agents/run
PROXY_SOCKET="$PWD/.agents/run/dbus-host-proxy"
rm -f "$PROXY_SOCKET"
xdg-dbus-proxy "${DBUS_SESSION_BUS_ADDRESS}" "$PROXY_SOCKET" \
  --filter \
  --talk=org.freedesktop.Notifications

Ready when you are — once it is running, tell me "done" and I will test
the socket.
MSG
}
```

If the socket exists but does not respond, print restart guidance and stop:

```bash
prompt_restart_proxy() {
  printf 'Host DBus proxy socket exists but is not responding at:\n  %s\n\n' "$PROXY_SOCKET"
  cat <<'MSG'
Please stop the host xdg-dbus-proxy process, remove the stale socket,
and restart it with the bootstrap command above.

Ready when you are — once it is running, tell me "done" and I will test
the socket.
MSG
}
```

Check whether the default notifications profile is usable:

```bash
check_notifications_proxy() {
  DBUS_SESSION_BUS_ADDRESS="unix:path=$PROXY_SOCKET" \
    gdbus call --session \
      --dest org.freedesktop.Notifications \
      --object-path /org/freedesktop/Notifications \
      --method org.freedesktop.Notifications.GetServerInformation >/dev/null 2>&1
}
```

Use the helpers like this. If a prompt function prints instructions, stop and
wait for the user before trying again:

```bash
if [ ! -S "$PROXY_SOCKET" ]; then
  prompt_start_proxy
  exit 1
fi

if ! check_notifications_proxy; then
  prompt_restart_proxy
  exit 1
fi

export DBUS_SESSION_BUS_ADDRESS="unix:path=$PROXY_SOCKET"
```

## Send a notification

Prefer `gdbus` because it is explicit and works without `notify-send`.
Choose notification fields deliberately:

- `APP_NAME`: the agent name, e.g. `"π"` or `"opencode"`.
- `SUMMARY`: a fixed, short summary of the session, ideally about 3 words.
  Prefer the session name when available; otherwise summarize the session
  content. If neither is meaningful yet, use the session start date in RFC3339
  format and revise it once the session has a clearer topic.
- `URGENCY`: `0` low, `1` normal, `2` critical.
- `BODY`: the detailed notification text.

```bash
WORKDIR="${PWD:-/stage}"
PROXY_SOCKET="$WORKDIR/.agents/run/dbus-host-proxy"
export DBUS_SESSION_BUS_ADDRESS="unix:path=$PROXY_SOCKET"

APP_NAME="π"
SUMMARY="DBus notification test"
URGENCY=0
BODY="Notification from the boxed agent."

gdbus call --session \
  --dest org.freedesktop.Notifications \
  --object-path /org/freedesktop/Notifications \
  --method org.freedesktop.Notifications.Notify \
  "$APP_NAME" 0 '' "$SUMMARY" "$BODY" [] "{'urgency': <byte $URGENCY>}" 5000
```

The call returns a notification id, for example `(uint32 145,)`.

## Optional profile: Open URI via Desktop Portal

Use this profile only when the user wants the agent to open host desktop URLs or
otherwise use the desktop portal. It extends the default notifications profile;
keep notifications enabled because OpenURI must normally be paired with a visible
notification.

Do not proceed with `open-uri` until this optional profile is verified. If it is
not verified, show the exact host bootstrap command below, stop, and wait for the
user to say it is running. The check below only tests an already-started proxy;
it is not a substitute for the user-side bootstrap.

Ask the user to restart the host proxy with this broader allowlist:

```bash
mkdir -p .agents/run
PROXY_SOCKET="$PWD/.agents/run/dbus-host-proxy"
rm -f "$PROXY_SOCKET"
xdg-dbus-proxy "${DBUS_SESSION_BUS_ADDRESS}" "$PROXY_SOCKET" \
  --filter \
  --talk=org.freedesktop.Notifications \
  --talk=org.freedesktop.portal.Desktop
```

Then verify portal access from the container. If the socket is missing or the
profile is not available, return control immediately and ask the user to start
the optional profile. When the user says it is running, run the check again. Do
not wait inside the tool call for the human to restart it.

Reuse the project-local socket variables from the default profile check:

```bash
WORKDIR="${PWD:-/stage}"
PROXY_SOCKET="$WORKDIR/.agents/run/dbus-host-proxy"
```

If OpenURI is not available, print the optional profile bootstrap command and
stop:

```bash
prompt_openuri_profile() {
  printf 'Desktop Portal / OpenURI is not available through:\n  %s\n\n' "$PROXY_SOCKET"
  cat <<'MSG'
Please restart the host proxy with the optional Desktop Portal profile
from the project root and keep it running:

mkdir -p .agents/run
PROXY_SOCKET="$PWD/.agents/run/dbus-host-proxy"
rm -f "$PROXY_SOCKET"
xdg-dbus-proxy "${DBUS_SESSION_BUS_ADDRESS}" "$PROXY_SOCKET" \
  --filter \
  --talk=org.freedesktop.Notifications \
  --talk=org.freedesktop.portal.Desktop

Ready when you are — once it is running, tell me "done" and I will test
OpenURI.
MSG
}
```

Check whether the optional OpenURI profile is usable:

```bash
check_openuri_profile() {
  DBUS_SESSION_BUS_ADDRESS="unix:path=$PROXY_SOCKET" \
    gdbus call --session \
      --dest org.freedesktop.portal.Desktop \
      --object-path /org/freedesktop/portal/desktop \
      --method org.freedesktop.DBus.Peer.Ping >/dev/null 2>&1 &&
  DBUS_SESSION_BUS_ADDRESS="unix:path=$PROXY_SOCKET" \
    gdbus introspect --session \
      --dest org.freedesktop.portal.Desktop \
      --object-path /org/freedesktop/portal/desktop 2>/dev/null |
    grep -q 'org.freedesktop.portal.OpenURI'
}
```

Use the helpers like this. If `prompt_openuri_profile` prints instructions, stop
and wait for the user before trying again:

```bash
if [ ! -S "$PROXY_SOCKET" ]; then
  prompt_openuri_profile
  exit 1
fi

if ! check_openuri_profile; then
  prompt_openuri_profile
  exit 1
fi

export DBUS_SESSION_BUS_ADDRESS="unix:path=$PROXY_SOCKET"
```

### Open a URL after notifying the user

Do not call `OpenURI` immediately unless the user explicitly asked for a silent
or immediate open. By default, use this protocol:

1. Send a notification with an action button.
2. Wait briefly for `org.freedesktop.Notifications.ActionInvoked`.
3. If the user clicked the expected action, call `OpenURI`.
4. If the user does not click before timeout, do nothing.

```bash
WORKDIR="${PWD:-/stage}"
PROXY_SOCKET="$WORKDIR/.agents/run/dbus-host-proxy"
export DBUS_SESSION_BUS_ADDRESS="unix:path=$PROXY_SOCKET"

APP_NAME="π"
SUMMARY="Open URL request"
URGENCY=0
BODY="Click to open Wikipedia in your browser."
URI="https://www.wikipedia.org/"
ACTION_KEY="default"
ACTION_LABEL="Open Wikipedia"
TIMEOUT_SECONDS=20
LOG="$(mktemp)"

gdbus monitor --session \
  --dest org.freedesktop.Notifications \
  --object-path /org/freedesktop/Notifications >"$LOG" 2>&1 &
MON_PID=$!
trap 'kill "$MON_PID" 2>/dev/null || true; rm -f "$LOG"' EXIT
sleep 0.3

NOTIFICATION_ID=$(gdbus call --session \
  --dest org.freedesktop.Notifications \
  --object-path /org/freedesktop/Notifications \
  --method org.freedesktop.Notifications.Notify \
  "$APP_NAME" 0 '' "$SUMMARY" "$BODY" \
  "['$ACTION_KEY', '$ACTION_LABEL']" \
  "{'urgency': <byte $URGENCY>}" \
  $((TIMEOUT_SECONDS * 1000)) |
  sed -n 's/^(uint32 \([0-9][0-9]*\),)$/\1/p')

for _ in $(seq 1 $((TIMEOUT_SECONDS * 2))); do
  if grep -q "ActionInvoked (uint32 $NOTIFICATION_ID, '$ACTION_KEY')" "$LOG"; then
    gdbus call --session \
      --dest org.freedesktop.portal.Desktop \
      --object-path /org/freedesktop/portal/desktop \
      --method org.freedesktop.portal.OpenURI.OpenURI \
      '' "$URI" "{'handle_token': <'openurl'>}"
    break
  fi
  sleep 0.5
done
```

Do not use other portal interfaces such as Screenshot, ScreenCast,
RemoteDesktop, Camera, Location, or FileChooser unless the user explicitly asks
for that capability and understands the host-side permission implications.

## Inspect the exposed interface

```bash
WORKDIR="${PWD:-/stage}"
PROXY_SOCKET="$WORKDIR/.agents/run/dbus-host-proxy"
export DBUS_SESSION_BUS_ADDRESS="unix:path=$PROXY_SOCKET"

gdbus introspect --session \
  --dest org.freedesktop.Notifications \
  --object-path /org/freedesktop/Notifications

gdbus call --session \
  --dest org.freedesktop.Notifications \
  --object-path /org/freedesktop/Notifications \
  --method org.freedesktop.Notifications.GetCapabilities
```

## Troubleshooting

- `No such file or directory`: the proxy socket is not mounted/visible or the
  user has not started `xdg-dbus-proxy`.
- `Connection refused`: stale socket; ask the user to remove it and restart the
  proxy.
- `ServiceUnknown` for portals or other services: expected with the default
  notification-only filter. For OpenURI, ask the user to restart the proxy with
  the optional Desktop Portal profile.
- `gdbus: command not found`: install GLib DBus tools on demand in the
  container, e.g. `apt-get update && apt-get install -y libglib2.0-bin` on
  Debian/Ubuntu. `dbus-bin` is optional debugging tooling, not required for the
  documented notification calls.
- Full desktop portal integration needs more than this DBus proxy; it also needs
  the relevant data channels, such as document portal FUSE or PipeWire sockets.

## Security notes

- This is intentionally narrower than mounting the host session bus directly.
- Only add `--talk=` rules that are needed for the task.
- Treat `org.freedesktop.portal.Desktop` as a broad opt-in capability, not a
  default. It exposes many portal interfaces beyond OpenURI.
- Do not bridge this socket to TCP unless the user understands the additional
  exposure; a project-local Unix socket is preferred.

## Metadata
* Author: thias <github.attic@typedef.net>, OpenAI gpt-5.5
* License: CC BY 4.0
* Version: 0.2
* Date: 2026-06-10
