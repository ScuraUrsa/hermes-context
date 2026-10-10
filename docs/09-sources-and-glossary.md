# 9. Sources and glossary

## Sources used for this snapshot (2026-10-10)

| Source | What it gave |
|---|---|
| Claude project "Hermes_local" files: `prod-stage-dev-design.md`, `cutover-and-self-upgrade-plan.md`, `vm-iac-tasks.md` (copied to [sources/](sources/)), `session-review/inventory.md`, `session-review/backup-findings.md`, `session-review/raw/cron_jobs.csv`, `notes/hermes-todo-2026-10-08.md`, `fidelity-result-2026-10-09.md`, `ci/github-actions-failures-2026-10-09.md`, `hbench/results-2026-10-09.md`, `guides/*`, `timeplane/audyt-prosb.md`, `youtube-summaries/findings.md` | design, decisions, current state |
| WOBI `master` and `iac/final` (`ansible/README.md`, roles, inventory, `features/registry.yaml`) | deployment facts |
| hermes-infra (`docs/self-upgrade-flow.md`, `hermes-host-daemon/`, `AUTO_MERGE_POLICY.md`) | host daemon, cutover |
| crewai-orchestrator (`docs/RUNBOOK-autonomy-loop.md`, delivery gate runbook, decision logs) | autonomy loop |
| hermes-agent fork (git log, `agent/provider_semaphore.py`, `hermes_cli/`) | Hermes internals |
| Notion: "Hermes Organized", "Baza wiedzy Hermesa", "Przegląd wiedzy: Hermes: konfiguracja i działanie", skill page "hermes-operations", "Przegląd sesji Hermesa" | operations, history |
| Project threads (VM lifecycle, GitHub Actions, coding tools benchmark, timeplane, phone access) | latest changes |

Not read for this snapshot: `D:\hermes\HERMES_MAP.md` on Filip's PC (the older system map and changelog),
the live VMs.

## Glossary

| Term | Meaning |
|---|---|
| prod / unity-orchestrator / hermes-vm-1 | the VM Filip talks to (192.168.56.107, Tailscale 100.66.185.39) |
| dev / hermes-dev | the work VM (192.168.56.108) |
| stage | throw-away VM built from candidate refs for upgrades |
| replica | a VM rebuilt from code + backup by the host daemon (standby until cutover) |
| cutover | switching prod to a rebuilt VM while keeping its identity (IP, Tailscale node, SSH keys) |
| active / standby | `hermes_role`: only active talks to Filip, uploads backups, joins Tailscale |
| WOBI | the provisioning repo (Ansible); name kept from the earlier "Unity AI Agent Orchestrator" |
| host daemon | `hermes-host-daemon`, the Windows service that manages VMs |
| profile | a separate Hermes home with its own persona/config (coder, reviewer, …) |
| kanban / card / t_xxxxxxxx | Hermes' task board and task ids |
| delivery gate | rule since 2026-10-08: task done = branch + PR + review task |
| planner / dispatcher | cron job that creates self-tasks / gateway loop that runs them |
| quota-governor, budget-governor | cron scripts that pick the work mode and enforce the USD budget |
| CI doctor | `gh_actions_watchdog.py`, turns red GitHub Actions into fix tasks |
| timeplane / tp | time-management add-on and its CLI |
| Honcho | self-hosted long-term memory service |
| BWS | Bitwarden Secrets Manager |
| gacct | wrapper to use one of Filip's 4 Google accounts |
| cc_run.py / gm_run.py / agent-run | runners for Claude Code / other coding agents on the VMs |
| feature registry | `WOBI/features/registry.yaml`, status of each feature (dev → prod-experimental → prod-stable) |
