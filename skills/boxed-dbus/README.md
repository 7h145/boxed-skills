# boxed-dbus skill

A small agent skill for using a host-created, filtered DBus proxy from inside a
containerized agent. The default setup exposes only desktop notifications via
`org.freedesktop.Notifications`.

It is designed for `*inabox` style containerized agent workflows such as
[piinabox](https://github.com/7h145/piinabox),
[ocinabox](https://github.com/7h145/ocinabox), or similar "agent in container"
setups.

## What is this?

`boxed-dbus` teaches the agent to use a project-local DBus proxy socket:

```text
.agents/run/dbus-host-proxy
```

The proxy is created on the host with
[`xdg-dbus-proxy`](https://github.com/flatpak/xdg-dbus-proxy), and the agent
connects to that filtered socket instead of the full host session bus. The agent
will provide the exact command and guide the user through starting or restarting
the proxy when needed.

## What it does

1. Checks for `.agents/run/dbus-host-proxy`.
2. Guides the user to start a host-side `xdg-dbus-proxy` if needed.
3. Uses the filtered socket as the agent's DBus session bus.
4. Sends notifications through `org.freedesktop.Notifications.Notify`.
5. Optionally, if the user opts in to the broader Desktop Portal profile, opens
   URLs through `org.freedesktop.portal.OpenURI.OpenURI`, normally after a
   notification action.

## Capability profiles

Quick profile names for prompts/status messages:

- `notifications`: default profile; only `org.freedesktop.Notifications`.
- `open-uri`: optional profile; adds `org.freedesktop.portal.Desktop` for host
  URL opening.

Example requests:

```text
Use boxed-dbus notifications to send a completion alert.
Use boxed-dbus open-uri to open https://www.wikipedia.org/ on my host.
```

### Default: notifications

The default profile exposes only:

```bash
--talk=org.freedesktop.Notifications
```

Use it for notifications and notification actions. This is the narrow, preferred
baseline.

### Optional: Desktop Portal / OpenURI

The optional profile adds:

```bash
--talk=org.freedesktop.portal.Desktop
```

Use it when the agent should open host desktop URLs, preferably after showing a
notification and waiting for the user to click an action button. This is broader
than notifications-only: the same portal service also exposes interfaces such as
Screenshot, ScreenCast, RemoteDesktop, Camera, Location, FileChooser, and more.
Portal backends may prompt the user, and grants may persist.

Do not enable this profile silently. The host user should deliberately restart
`xdg-dbus-proxy` with the broader allowlist.

## Platform requirements

This workflow assumes a Linux desktop host with a freedesktop-compatible session
DBus and notification service. The host must be able to run `xdg-dbus-proxy`
from an environment where `DBUS_SESSION_BUS_ADDRESS` points at the real desktop
session bus.

It is not generally applicable to Windows/macOS hosts or WSL-backed Docker/Podman
setups unless the user has explicitly provided a compatible DBus/notification
bridge. WSL may have its own Linux DBus environment, but that does not usually
mean it can reach the user's actual desktop notification service.

## Security model

This is intentionally narrower than mounting the full host
`DBUS_SESSION_BUS_ADDRESS` into the container. The proxy is still a capability:
only add rules for interfaces the agent actually needs.

`org.freedesktop.portal.Desktop` is useful but broad. It should be treated as an
opt-in profile rather than part of the default. Some portal permissions are
stored persistently by xdg-desktop-portal, commonly under:

```text
~/.local/share/flatpak/db
```

Host-side Flatpak tooling can inspect or reset dynamic portal permissions, for
example:

```bash
flatpak permissions
flatpak permissions screenshot screenshot
flatpak permission-remove screenshot screenshot
flatpak permission-reset APP_ID
```

Desktop environments differ: GNOME/KDE may expose some portal permissions in
privacy or app settings, while wlroots-style setups may not have a polished UI.

Full desktop portal integration may need more than this DBus proxy; it can also
need relevant data channels, such as document portal FUSE or PipeWire sockets.
This skill currently documents OpenURI only as the optional portal workflow.

## Install

Install this skill through your agent harness, or copy this skill directory into
wherever your harness loads skills from.

Pi example:

```bash
pi install git:github.com/7h145/boxed-skills
```

[npm skills](https://www.npmjs.com/package/skills) example:

```bash
npx skills add 7h145/boxed-skills
```
