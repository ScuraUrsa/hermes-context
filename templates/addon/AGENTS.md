# AGENTS.md: hermes-<name>

<!-- Template for an add-on repo. Replace <...>. Keep it short. -->

## What this is

<One paragraph: what the add-on does for Filip and how Hermes uses it.>

Part of Hermes, Filip's personal assistant. Context for Hermes itself:
https://github.com/ScuraUrsa/hermes-context (read its AGENTS.md first).

## How Hermes uses it

- Skill: `skill/SKILL.md` (installed to `~/.hermes/skills/<category>/<name>`)
- CLI: `<cmd>` (installed to `~/.local/bin/<cmd>`)
- Cron jobs: `<job>` every <n> min, no-agent | agent
- Service: `<name>.service` on <port>, bound to <127.0.0.1 | host-only | Tailscale>
- State: `~/.hermes/autonomy/<name>/` (in the nightly backup)
- Runs on: prod | dev (active only for anything that reaches Filip's accounts)
- Secrets (Bitwarden key names only): `<KEY_NAME>`

## Deploy

WOBI role `ansible/roles/<name>` pins this repo at `<name>_version`. Bump the pin in a WOBI PR after a
release here. Feature registry id: `<name>`.

## Develop

```bash
python -m venv .venv && . .venv/bin/activate && pip install -e .[dev]
ruff check . && pytest -q        # tests run offline
```

## Rules

PRs only, no secrets, decision log in `docs/decision_log_<topic>.md`, Polish for Filip-facing text.
