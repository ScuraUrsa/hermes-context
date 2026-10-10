# 2. Architecture

## Physical picture

```
Filip's Windows laptop (32 GB RAM, VirtualBox, D:\hermes = local workspace with clones and notes)
│
├── hermes-host-daemon (Python, :9143)  creates/destroys VMs, builds replicas, runs approved cutovers
│
├── host-only network 192.168.56.0/24 (VirtualBox DHCP + reservations; host = 192.168.56.1)
│   ├── PROD   192.168.56.107  identity "unity-orchestrator", Tailscale hermes-vm-1 / 100.66.185.39
│   ├── DEV    192.168.56.108  "hermes-dev" (no Tailscale yet)
│   └── STAGE  192.168.56.110  built on demand from the `stage` refs, then deleted (design, not live yet)
│
└── Tailscale tailnet: phone (Termux, HA app), laptop, prod VM
```

Since the cutover of 2026-10-09 the prod VM is a machine *rebuilt from code* (its hostname is
`hermes-replica-66641d16`, it runs with the identity of `unity-orchestrator`: same IP, Tailscale node,
SSH host keys, Syncthing device). The original hand-built VM is gone.

## What runs where (prod / dev / stage)

Design: [`sources/prod-stage-dev-design.md`](sources/prod-stage-dev-design.md)
(copied from the project files). Summary:

| | **prod** (always on) | **dev** (always on) | **stage** (on demand) |
|---|---|---|---|
| Purpose | talks to Filip, acts for him | code work: repos, worktrees, kanban workers | candidate for the next prod |
| Hermes | gateway, CLI, Filip's cron jobs (timeplane, inbox, google-token-check), memory | worker mode: kanban dispatcher, profiles, Claude Code, autonomy cron (planner, budget/quota governor, CI watchdog, pr-merge-watch); no chat platforms | full prod setup, no chat platforms |
| Kanban board | client (`kanban_bridge` client → dev API :8787, snapshot `kanban-dev.db` every minute) | owner (`kanban.db`, `kanban-api.service`) | test copy |
| Honcho memory | yes (source of truth, :8100) | reads prod's Honcho read-only | restored copy |
| Home Assistant + Whisper + Piper, notification bridge, timeplane intake, Syncthing | yes | no | yes (for tests) |
| Repos and worktrees | none | all | none |

Only **work orders** cross from prod to dev (kanban API). Dev pushes back task status / "please test" events.
Each VM reads only the Bitwarden secrets its role needs; nothing secret is copied between VMs.

## Software inside a VM

```
systemd
├── hermes-gateway.service          Hermes gateway (API server, cron scheduler, kanban dispatcher tick 60 s)
├── hermes-backup.timer (03:00)     ~/.hermes/scripts/hermes_backup.sh → rclone crypt → Google Drive
├── hermes-curator.timer            Hermes' session/skill curator
├── ollama.service                  local Ollama, only for Honcho embeddings (honcho-embed:qwen3-0.6b)
├── timeplane-intake.service        (prod) HTTP intake for HA buttons / phone signals
├── kanban-api.service              (dev) kanban board API for prod
├── syncthing@ubuntu                (prod) Obsidian vault
├── tailscaled                      (prod)
└── docker
    ├── honcho (api, deriver, postgres, redis, telemetry-sink)   ~/honcho
    ├── homeassistant + wyoming-whisper + wyoming-piper          ~/homeassistant   (prod)
    ├── hermes-notification-bridge :8655                          (prod)
    └── crewai stack (worker, redis, postgres, litellm)           RETIRED, off by default

~/.hermes                      Hermes home: config.yaml, .env, auth.json, SOUL.md, memories/, skills/,
                               profiles/<name>/, cron/jobs.json, kanban.db, state.db, autonomy/, scripts/
~/.hermes/hermes-agent         Hermes code: upstream NousResearch @ a6ede70 (v0.20.0), pinned by WOBI hermes_ref; venv
~/.local/bin/hermes            Filip's launcher (checks Ollama keys, then starts Hermes)
~/.claude-code                 Claude Code config dir (CLAUDE_CONFIG_DIR), never ~/.claude
```

## Hermes concepts you need

- **Gateway**: the long-running process (chat platforms, API server, cron, kanban dispatcher).
- **Profiles**: separate Hermes homes under `~/.hermes/profiles/<name>` with their own SOUL.md, config,
  skills and `state.db`: `main` (Filip's), `architect`, `coder`, `maintainer`, `researcher`, `reviewer`,
  `writer`, `copilot-pro`, `copilot-archive`. Kanban tasks are assigned to a profile.
- **Skills**: folders with `SKILL.md` (agentskills.io format) in `~/.hermes/skills/<category>/<name>`.
  Shared skills for all coding agents: `~/.agents/skills` (WOBI role `coding_agents`).
- **Plugins**: Python plugins loaded by config (e.g. `honcho`, `timeplane`).
- **Cron**: Hermes' own scheduler (`hermes cron list`), jobs in `~/.hermes/cron/jobs.json`; "no-agent"
  jobs run a script without an LLM.
- **Kanban**: Hermes' task board (`hermes kanban list|create|show|complete`), SQLite `kanban.db`. Workers
  run in git worktrees (`wt/t_<id>` branches) of the target repo.
- **Memory**: `memories/MEMORY.md` (≤2200 chars) + `USER.md` (≤1375 chars) in every prompt; Honcho for
  long-term recall (workspace `filip`, aiPeer `hermes`, one shared `~/.hermes/honcho.json`).

## Models and providers

| Use | Provider / model | Notes |
|---|---|---|
| Main chat | Ollama Cloud `deepseek-v4-pro` (default switched to DeepSeek V4.1 Flash for some paths on 2026-10-05; check `config.yaml`) | two Ollama Pro keys in a credential pool (`fill_first`), kept honest by `ollama_keys.py` |
| Profiles / auxiliary tasks | Ollama Cloud `deepseek-v4.1-flash` | `deepseek-v4-flash` was retired 2026-09-25 (HTTP 410) |
| Coding | Claude Code CLI (Filip's subscription) via `~/.hermes/scripts/cc_run.py`; also Zed headless agent via `agent-run` (WOBI #6, dev) and `gm_run.py` | exit 75 = Claude quota used → block the task, never fall back to coding on Ollama |
| Honcho reasoning | deepseek-v4.1-flash via Ollama Cloud; embeddings local (`honcho-embed`) | |
| Speech | Whisper small-int8 (pl), Piper `pl_PL-gosia-medium` | in HA compose |

Benchmark of coding frameworks × providers on Hermes tasks (2026-10-09): Claude Code + opus solved 7/7,
sonnet 6/7 fastest; Ollama-hosted models 5–6/7. See `hermes-enhancements` PR #6 (hbench).

## Data, state and backups

- **State** (cannot be rebuilt from Git): `~/.hermes` DBs and memories, Honcho Postgres, HA `.storage`,
  Syncthing identity, Tailscale node key, SSH host keys, dirty worktrees.
- **Nightly backup** (format 3): tar+zstd of `~/.hermes` without code/caches, consistent SQLite snapshots,
  Honcho dump, HA config, identities (`extra/identity.tar`) and unpushed work (`extra/work.tar`: git bundles
  + patches); encrypted with rclone crypt; uploaded to the first Google Drive account with space, in
  order praktyki → fjkazmierczak → koalamis6 → kazmierczakfilipjan, folder `hermes-backups-encrypted`,
  7 kept. Status in `~/.hermes/autonomy/backup_status.json`.
- **Everything else is code**: WOBI Ansible + pinned refs + Bitwarden secrets.
