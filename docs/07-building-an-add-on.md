# 7. Building a separate add-on product for Hermes

This is the path for an agent in another project that builds something new *around* Hermes (a new
capability, integration or app) without having to read every Hermes repo. The reference example is
**timeplane** (time management): its own repo `ScuraUrsa/hermes-timeplane`, a CLI (`tp`), a Hermes plugin
and skill, cron jobs, an HTTP intake service, Notion pages, and a small WOBI role that deploys it.

## 1. Decide the shape

| You are building… | Integration point | Example |
|---|---|---|
| Knowledge/procedure Hermes should follow | **Skill** (`SKILL.md`, agentskills.io format) | `hermes-operations`, `hermes-timeplane` skill |
| Tools Hermes calls with structured arguments | **Hermes plugin** (Python, enabled in `config.yaml`) | `plugins/timeplane`, `honcho` |
| A command-line tool Hermes and Filip both use | **CLI** in `~/.local/bin` + a skill that teaches it | `tp`, `gacct`, `phone`, `agent-run` |
| Something that runs on a schedule | **Hermes cron job**; prefer *no-agent* (script only, no LLM) jobs, use an LLM only when needed | `timeplane-tick`, `google-token-check` |
| Something that needs to *ask Hermes* (a UI, HA, another app) | **Hermes API server** `127.0.0.1:8642/v1` (OpenAI-compatible, model `hermes`, key `HERMES_API_SERVER_KEY`); local to prod only | HA voice pipeline |
| A long-running service | **systemd unit or Docker Compose service**, bound to localhost / host-only / Tailscale only | `timeplane-intake`, notification bridge |
| Work Hermes should do in the background | **kanban task** for a profile (`hermes kanban create ...`) | YouTube archive tasks |
| Something Filip sees or taps | Notion page/database, Home Assistant entity/button, phone notification | "🧭 Hermes — Plan dnia", HA break buttons |
| A new provider/model/coding tool | WOBI role + runner script like `cc_run.py` | `coding_agents` role, `zed_run.py` |

Keep the add-on usable by a weaker model: a CLI with clear subcommands and a short skill beats a long prompt
(timeplane was designed so DeepSeek can drive it; Claude only for architecture).

## 2. Repository

- New private repo `ScuraUrsa/hermes-<name>` (or a directory in `hermes-enhancements` for experiments).
- Contents: code, offline tests, CI (ruff + pytest at minimum), `README.md`, `AGENTS.md` (copy
  [`templates/addon/AGENTS.md`](../templates/addon/AGENTS.md)), a `skill/` folder, a decision log.
- Layout that WOBI can install by copying paths (timeplane pattern): `scripts/`, `plugin/<name>/`,
  `skill/`, optional `ha/`, `systemd/`.
- No secrets. Read them at run time from env or Bitwarden by key name; document the key names.

## 3. Runtime contract on the VM

- User `ubuntu`, home `/home/ubuntu`. Hermes home `~/.hermes`; code in `~/.hermes/scripts/<name>`,
  state in `~/.hermes/autonomy/<name>` (inside `~/.hermes` → included in the nightly backup automatically).
- Python: use Hermes' venv `~/.hermes/hermes-agent/venv/bin/python` or your own venv; don't rely on
  `PATH` inside cron/systemd (a bare `yt-dlp` call broke a pipeline once).
- Scripts must resolve the real home via `pwd`/passwd, not `$HOME` (workers run with
  `HOME=~/.hermes/profiles/<p>/home`).
- Decide the VM: Filip-facing (chat, phone, HA, Google writes) → **prod**; code work, heavy jobs, builds →
  **dev**. Anything that writes to Filip's accounts must run only when `hermes_role == active`.
- Talking to Filip: return text from the cron job (delivered by Hermes), write to Notion, or push through
  HA / the notification bridge. Don't invent a new channel.
  Never call `hermes send -t matrix` while the gateway runs: it reuses the gateway's Matrix device and
  kills its token (see [05-operations](05-operations.md#matrix)).
- Memory: read Honcho if you need user context; only prod writes Filip's memory.
- Cost: every LLM call by Hermes is counted against the self-work budget ($4/day, $24/week in `budget.json`). Coding through
  Claude Code only. Prefer deterministic scripts.

## 4. Deploy through WOBI (no hand edits on the VMs)

1. Add a role `ansible/roles/<name>/` modeled on `roles/timeplane`: check out your repo at a **pinned
   commit** (`<name>_version`), copy files into place (never overwrite differing live files silently),
   install units/launchers, read secrets with `lookup('bws_secret', 'KEY')` + `no_log: true`, start services
   only on `active`.
2. Add `inventory/group_vars/all/<name>.yml` with `<name>_enabled: true|false`; per-host overrides in
   `host_vars/<host>/<name>.yml` (dev turns Filip-facing roles off in `toggles.yml`).
3. Add the role to `site.yml` (shared file; ask in the PR) and register cron jobs, if any, in
   `hermes_role`'s paused/resumed lists for each host.
4. Add an entry to `features/registry.yaml` with `status: dev` (template in
   [`templates/addon/registry-entry.yaml`](../templates/addon/registry-entry.yaml)).
5. Test on dev or a replica: `sudo bash ansible/pull.sh --as <host> --role standby --ref <branch> --tags <name> --check`,
   then without `--check`, twice (second run: no changes).
6. PR to WOBI; Filip approves. Prod gets it via stage/cutover or `pull.sh --role active`; the registry entry
   moves to `prod-experimental` with a review gate 7 days later, then `prod-stable` or `retired`.

## 5. Checklist before asking Filip

- [ ] Own repo with README, AGENTS.md, tests green in CI, no secrets (`git grep` for tokens).
- [ ] Skill that tells Hermes when and how to use it (and when not to).
- [ ] WOBI role, idempotent, `ansible-lint` clean, toggled, pinned.
- [ ] Feature registry entry.
- [ ] State lives under `~/.hermes` (backed up) or the backup was extended.
- [ ] Nothing listens publicly; no new chat bot; no OAuth granted by the agent.
- [ ] Notion page in Polish describing how Filip uses it (short, like `guides/zarzadzanie-czasem.md`).
- [ ] Decision log with what was decided and why.
