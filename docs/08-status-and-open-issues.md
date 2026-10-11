# 8. Status and open issues (snapshot 2026-10-11)

This page goes stale fastest. Check the sources before acting on it.

## Recently done

- Whole VM described as code in WOBI; first real cutover to a rebuilt VM on 2026-10-09 (identity kept).
- Dev VM split out (2026-10-09): repos, worktrees and the kanban board moved to `hermes-dev`.
- Timeplane v2 live; Home Assistant voice pipeline in Polish; Honcho memory working.
- Delivery gate (PR + review task) since 2026-10-08; CI fixed in most repos on 2026-10-09/10;
  `pr-merge-watch` repaired 2026-10-11.
- Phone ↔ Hermes end to end (Termux aliases `h` / `c`, phone control) 2026-10-09; Matrix server up 2026-10-09;
  contacts CRM live 2026-10-11.
- Session quality review of all ~1540 Hermes sessions in progress (Notion "Przegląd sesji Hermesa").

## Known problems

| Problem | Where | Note |
|---|---|---|
| Prod nightly backup failed 2026-10-10 ("snapshot missing kanban.db") | prod, `hermes_backup.sh` | board moved to dev; backup script must accept the snapshot / dev board |
| Self-work budget not holding | `budget.py`, governors | overspent on 2026-10-08 |
| Maintenance tasks repeat without effect | planner | ~2/3 of maintenance runs are repeats |
| Workers end rc=0 without completing the card; stale blocked cards | kanban | close with CLI |
| WOBI `iac/final` not merged to `master` | WOBI | the daemon builds from `iac/final` |
| VMs run Hermes v0.20.0, upstream far ahead | WOBI `hermes_ref` | upgrade via replica/stage; fork PR #5 |
| No chat platform on the gateway | prod | Matrix proposed |
| Dev has no Tailscale; stage pipeline needs a Tailscale OAuth client | Filip | |
| Home Assistant listens on 0.0.0.0:8123 | prod | should be Tailscale/host-only only |
| Hermes runs with approvals off (YOLO) | prod config | safety rules are only in prompts and skills |
| Honcho spends Ollama credit outside the budget | prod | ~83% of its recall calls come from kanban workers |
| Matrix server and contacts CRM are not in WOBI or the backup | prod | a rebuild would lose them |
| GitHub Actions does not start on private repos (billing / spending limit) | GitHub account | seen on hermes-contact-crm 2026-10-11 |

## Secrets that leaked (rotation is Filip's decision)

- `hermes-config-backup`: `.env`, `auth.json`, `state.db` (plain-text tokens incl. a Bitwarden access
  token, Google OAuth refresh tokens, Ollama and Tailscale keys), an old skill with a Bitwarden token.
- `hermes-infra` history: commit `f772abc` added a `.env` with the Bitwarden bootstrap token and provider keys.
- `terraforge-modules`: `ssh/id_rsa`.
- VM git remotes contain GitHub PATs.
Details and proposed actions: `session-review/backup-findings.md` in the project files, Notion
"Przegląd sesji Hermesa" → Backup section. Never copy these values anywhere.

## Pending decisions for Filip

- Review gates in `features/registry.yaml` (timeplane 2026-10-15; voice, bridge, backup, cutover, dev VM 2026-10-16).
- Merge of WOBI #6 (Zed/agent-run), #7 (CI doctor budget), hermes-agent #5 (upstream merge).
- Rotation of the leaked secrets; archiving `hermes-config-backup`.
