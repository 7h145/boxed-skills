# boxed-skills

Agent skills for containerized `*inabox` coding-agent workflows, such as
[piinabox](https://github.com/7h145/piinabox) and
[ocinabox](https://github.com/7h145/ocinabox).

## Included skills

- [`boxed-tmux`](./skills/boxed-tmux/) — start and control a project-local tmux
  server/socket for long-running commands, dev servers, watchers, debugging
  shells, and Pi subagents.
- [`boxed-marimo`](./skills/boxed-marimo/) — start a marimo notebook inside
  boxed tmux and pair with the user through `marimo-pair`.
- [`boxed-dbus`](./skills/boxed-dbus/) — use a host-created, filtered
  `xdg-dbus-proxy` socket for narrow desktop DBus access such as notifications.

## Install

Follow the documentation for your agent harness. For example, for the
[Pi coding agent](https://pi.dev), see the
[skills documentation](https://pi.dev/docs/latest/skills) and
[package management documentation](https://pi.dev/docs/latest/packages).

Pi example: install from GitHub as a Pi package:

```bash
pi install git:github.com/7h145/boxed-skills
```

Or copy the skill directories into wherever your agent harness loads skills from.

## Usage

Follow the documentation for your agent harness. For example, for the
[Pi coding agent](https://pi.dev), see the
[skills documentation](https://pi.dev/docs/latest/skills).

Pi example: skills are loaded on demand when relevant. You can also invoke them
explicitly:

```text
/skill:boxed-tmux start a dev server in tmux
/skill:boxed-marimo start a notebook_mo.py notebook
/skill:boxed-dbus send me a desktop notification
```

See each skill directory for detailed instructions and examples.
