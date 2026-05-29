# boxed-marimo skill

`boxed-marimo` starts a marimo notebook inside an agent-owned tmux session and
then pairs with the user through the `marimo-pair` skill.

It is designed for `*inabox` style containerized agent workflows such as
[piinabox](https://github.com/7h145/piinabox),
[ocinabox](https://github.com/7h145/ocinabox), or similar "agent in container"
setups.

## Requirements

- `boxed-tmux` skill from this package
- `marimo-pair` skill. If it is missing, the agent should ask before installing
  it project-locally/non-interactively. For example:

```bash
npx -y skills add marimo-team/marimo-pair --agent opencode --skill marimo-pair -y
```

Adjust `--agent` for your harness when needed.

## What it does

1. Starts marimo in boxed tmux with `uvx marimo@latest edit ... --headless`.
2. Reports the marimo URL to the user.
3. Waits for the user to open the URL.
4. Connects via `execute-code.sh` and sends a ready toast.

## Install

Install this skill through your agent harness, or copy this skill directory into
wherever your harness loads skills from.

Pi example:

```bash
pi install git:github.com/7h145/boxed-skills
```
