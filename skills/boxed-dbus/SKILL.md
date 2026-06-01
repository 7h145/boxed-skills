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

## Naming

Default project-local proxy socket:

```bash
WORKDIR="${PWD:-/stage}"
PROXY_SOCKET="$WORKDIR/.agents/run/dbus-host-proxy"
```

`boxed-dbus` is the generic skill name because the same pattern can be reused for
other allowlisted DBus interfaces. For notification-only setups, names such as
`boxed-notify` or `dbus-notifications-proxy` are also reasonable, but this skill
uses `boxed-dbus` and `.agents/run/dbus-host-proxy` by convention.

## User-side bootstrap

The proxy must be started by the human on the host, from the project root, so it
can access the host desktop session bus:

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
bus access.

## Agent check / bootstrap prompt

Before using the proxy, verify that the socket exists and responds. Prefer
`gdbus` for all checks and calls. If `gdbus` is missing, install it on demand
inside the container; on Debian/Ubuntu it is provided by `libglib2.0-bin`:

```bash
apt-get update && apt-get install -y libglib2.0-bin
```

If the proxy socket is missing or inactive, instruct the user to run the
user-side bootstrap above.

```bash
WORKDIR="${PWD:-/stage}"
PROXY_SOCKET="$WORKDIR/.agents/run/dbus-host-proxy"

if [ ! -S "$PROXY_SOCKET" ]; then
  cat <<MSG
No active host DBus proxy socket found at:
  $PROXY_SOCKET

Please run this on the host from the project root and keep it running:

  mkdir -p .agents/run
  PROXY_SOCKET="\$PWD/.agents/run/dbus-host-proxy"
  rm -f "\$PROXY_SOCKET"
  xdg-dbus-proxy "\${DBUS_SESSION_BUS_ADDRESS}" "\$PROXY_SOCKET" --filter --talk=org.freedesktop.Notifications
MSG
  exit 1
fi

export DBUS_SESSION_BUS_ADDRESS="unix:path=$PROXY_SOCKET"
gdbus call --session \
  --dest org.freedesktop.Notifications \
  --object-path /org/freedesktop/Notifications \
  --method org.freedesktop.Notifications.GetServerInformation >/dev/null
```

If the final `gdbus` command fails, treat the proxy as inactive/stale and ask the
user to restart it with the bootstrap command.

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
  notification-only filter.
- `gdbus: command not found`: install GLib DBus tools on demand in the
  container, e.g. `apt-get update && apt-get install -y libglib2.0-bin` on
  Debian/Ubuntu. `dbus-bin` is optional debugging tooling, not required for the
  documented notification calls.
- Full desktop portal integration needs more than this DBus proxy; it also needs
  the relevant data channels, such as document portal FUSE or PipeWire sockets.

## Security notes

- This is intentionally narrower than mounting the host session bus directly.
- Only add `--talk=` rules that are needed for the task.
- Do not bridge this socket to TCP unless the user understands the additional
  exposure; a project-local Unix socket is preferred.

## Metadata
* Author: thias <github.attic@typedef.net>, OpenAI gpt-5.5
* License: CC BY 4.0
* Version: 0.1
* Date: 2026-06-01
