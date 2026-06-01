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

## Security model

This is intentionally narrower than mounting the full host
`DBUS_SESSION_BUS_ADDRESS` into the container. The proxy is still a capability:
only add rules for interfaces the agent actually needs.

Full desktop portal integration would need more than a DBus proxy; it also
needs the relevant data channels, such as document portal FUSE or PipeWire
sockets. This is currently out of scope of this skill.

## Install

Install this skill through your agent harness, or copy this skill directory into
wherever your harness loads skills from.

Pi example:

```bash
pi install git:github.com/7h145/boxed-skills
```
