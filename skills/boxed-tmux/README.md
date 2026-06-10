# boxed-tmux skill

A small agent skill for running long-lived commands in tmux via a project-local
socket. Multiple agents in the same project share the socket but use separate
tmux sessions.

It is designed for `*inabox` style containerized agent workflows such as
[piinabox](https://github.com/7h145/piinabox),
[ocinabox](https://github.com/7h145/ocinabox), or similar "agent in container"
setups.

## What is this?

`boxed-tmux` teaches the agent to start and control tmux via a project-local
socket, with one tmux session per agent:

```text
.agents/run/tmux-socket
session: agent-<id>
```

Use it for dev servers, file watchers, marimo notebooks, debugging shells, and
tmux subagents that should run asynchronously without blocking the main agent.

## Why does it exist?

Containerized coding agents should not need to control a host tmux session just
to run background work. In `*inabox`, this skill keeps tmux work on the agent
side of the container boundary while still allowing a human to attach for
visibility/debugging when needed:

```bash
tmux -S .agents/run/tmux-socket list-sessions
tmux -S .agents/run/tmux-socket attach -t agent-<id>
```

## Runtime model

There is one project-local tmux socket. The first agent that bootstraps tmux
creates the tmux server and becomes the **tmux host** for the project. Other
agents use their own tmux sessions on that same server:

```text
.agents/run/tmux-socket     # shared project socket
agent-<id>                  # one session per agent
```

Important consequence: tmux windows are spawned by the tmux server, so all tmux
workloads run in the tmux host agent's environment/container. If another agent
starts a dev server or marimo through this socket, that process keeps running
after that requesting agent exits, as long as the tmux host stays alive.

If the tmux host dies, the tmux server and its child processes die too. A later
agent can bootstrap tmux again and become the new tmux host, but old background
work will need to be restarted.

This leads to a useful project pattern:

- one central/tmux-host agent owns background work for the project
- other user-started agents or tmux subagents can request work through the shared
  socket
- humans attach only for inspection/debugging

## Host-owned escape hatch

The tmux socket is a capability. Using a socket means trusting and executing in
the tmux server's runtime environment. If you deliberately start a tmux server
with a socket in a shared writable path, agents that connect to it can ask that
tmux server to spawn processes.

Example host-owned socket:

```bash
mkdir -p .agents/run
tmux -S .agents/run/tmux-socket new-session -s host-tmux
```

In that mode, tmux workloads run on the host, not inside the agent container.
This can be useful for debugging host-specific issues, GUI/browser tools, host
credentials, devices, or namespaces. It also intentionally breaks the normal
container boundary: agents can cause host-side process execution through tmux.
Use the standard boxed socket unless the user explicitly asks for another one;
nonstandard sockets may intentionally change the execution boundary.

Don't do this at home, kids!  If you are not sure about the consequences
described here, do not create a host-owned tmux socket in the shared path.

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
