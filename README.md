# Marauder CLI skill

Agents use this skill to drive the Marauder account CLI (`python -m app.cli`) with an Agent key. Same key as MCP. No broker orders.

## Install

Copy `SKILL.md` into your agent's skills directory:

- Hermes: `~/.hermes/skills/marauder-cli/SKILL.md`
- Cursor: `.cursor/skills/marauder-cli/SKILL.md`
- Other agents: the folder your agent loads as `marauder-cli`

Or point the agent at the raw file:

https://github.com/Balllvin/marauder-cli/blob/main/SKILL.md

## Use

From a Marauder `backend/` checkout, with the project venv:

```bash
python -m app.cli whoami
python -m app.cli --help
```

Login, if you do not already have `~/.config/marauder/credentials`:

```bash
printf '%s\n' "$MARAUDER_AGENT_KEY" | python -m app.cli login --base-url https://notebook-marauder.com --key-stdin
```

Do not put the key in the command arguments.

The skill in this repo is the public copy. The same file lives in [Balllvin/marauder](https://github.com/Balllvin/marauder) at `.agents/skills/marauder-cli/SKILL.md` and `.cursor/skills/marauder-cli/SKILL.md`.
