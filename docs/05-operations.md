# 5. Operations

The authoritative runbook on the VMs is the Hermes skill **`hermes-operations`** (copy in Notion:
"Skill hermes-operations: zasady działania Hermesa"). This page is the summary an outside agent needs.

## Cron jobs (Hermes scheduler, state 2026-10-10)

| Job | Every | Runs on | What it does |
|---|---|---|---|
| `autonomy-planner` | 30 min | dev | LLM planner: reads roadmap + compact backlog (`planner_context.py`), creates small self-tasks on the kanban board; skips the LLM when budget is spent, mode is idle, or ≥5 self-tasks are open |
| `autonomy-budget-governor` | 15 min | dev | prices sessions in USD (`budget.py`), parks self-tasks when the cap is hit |
| `quota-governor` | 10 min | dev | `quota_governor.py`: decides mode `coding` / `non_coding` / `idle` in `~/.hermes/autonomy/quota.json`, holds tasks (`NO_OLLAMA`, `BUDGET`, `CLAUDE_QUOTA`), runs `ollama_keys.py check --apply` |
| `gh-actions-watchdog` ("CI doctor") | 15 min | dev | finds red GitHub Actions in ScuraUrsa repos, opens fix tasks (no duplicates, respects budget) |
| `pr-merge-watch` | 10 min | dev | `delivery_gate.py reconcile`: closes review tasks and cards after merge (erroring on 2026-10-10: missing venv path) |
| `autonomy-user-inbox` | 5 min | prod | turns `~/.hermes/autonomy/inbox.md` into priority-100 cards |
| `google-token-check` | daily | prod | silent if OK; prints `GOOGLE AUTH BROKEN` / `BACKUP STALE` otherwise |
| `timeplane-day` / `-tick` / `-fast` / `-evening` / `-informer` / `-review` | 05:00 / 10 min / 1 min / 19:30 / 03:30 / 16:00 UTC | prod | time management (see timeplane repo) |

Which job runs on which VM is enforced by WOBI role `hermes_role` (`hermes_role_cron_paused` /
`hermes_role_cron_resumed` in host_vars).

## The autonomy loop

```
~/.hermes/autonomy/roadmap.md (goals C1..C4) ─► autonomy-planner (30 min) ─► kanban backlog
   ─► dispatcher in the gateway (tick 60 s) ─► worker profile in a git worktree (wt/t_<id>)
   ─► coding via Claude Code (cc_run.py) ─► branch + PR + a review task (delivery gate, since 2026-10-08)
   ─► reviewer profile ─► merge ─► decision log in the repo ─► Notion "Baza wiedzy Hermesa"
```

- Priorities: Filip's tasks 100, self-tasks 0–10. Budget caps apply only to self-work (sources `cron`,
  `kanban`): $4/day, $24/week in `budget.json` (USD, priced with Ollama per-1M rates; was $2/$10 until early
  October, check the file). Filip's sessions are never capped. On 2026-10-08 self-work overspent the cap
  ($11.72 in a day), so treat the cap as unreliable.
- Worker limits: `kanban.max_in_progress 2`, per profile 1, `max_spawn 1`; `agent.max_turns` coder 40,
  others 60. Plan small tasks (one deliverable, ≤ ~30 steps) instead of raising limits.
- `--skill` takes skill **names**, never category names.
- Known failure modes: worker exits rc=0 without `kanban_complete` / `kanban_block`; stale `blocked` cards
  whose work is already merged (close with `hermes kanban complete` from the CLI, worker scope cannot);
  concurrent tasks bumping the same index file (DECISION-LOG-INDEX) → red master; repetitive maintenance
  tasks with no new effect (~2/3 of maintenance runs). The session review in Notion grades all of this.
- Since 2026-10-04 to 2026-10-10: ~1050 kanban tasks (997 done, 22 blocked), 890 created by the planner.

## Health checks (on a VM)

```bash
systemctl is-active hermes-gateway
cat ~/.hermes/autonomy/quota.json                 # mode, keys, Claude windows, held tasks
python3 ~/.hermes/scripts/budget.py status
python3 ~/.hermes/scripts/cc_run.py --status
hermes auth list ollama-cloud
hermes cron list
hermes kanban list --status blocked
hermes sessions stats
hermes memory status                              # Honcho plugin available?
cat ~/.hermes/autonomy/backup_status.json
cat /var/lib/wobi/last-run.json                   # which WOBI ref/role was applied last
```

Pause / resume all autonomous work: `hermes pause` / `hermes resume` (cutovers use this).

## Claude Code on the VMs

```bash
python3 ~/.hermes/scripts/cc_run.py --cwd "$HERMES_KANBAN_WORKSPACE" --task-id <id> -- "<precise spec>"
#   --model opus | --resume <session> | --probe | --status
#   exit 0 ok · 1 error · 75 CLAUDE_QUOTA (stop, kanban_block, no Ollama fallback) · 77 not logged in
```

Runs are logged to `~/.hermes/autonomy/claude_runs.jsonl`; the session id goes into `kanban_complete`
metadata as `claude_session` so Filip can `claude --resume <id>`. `CLAUDE_CONFIG_DIR=~/.claude-code`.
WOBI #6 (dev) adds `agent-run --agent zed|claude ...` and shared skills in `~/.agents/skills`.

## Google

`gacct <alias> <google_api.py args>` (e.g. `gacct koalamis6 gmail search "is:unread" --max 10`);
Drive via `rclone <alias>:`. Tokens in `~/.hermes/google-accounts/<alias>/google_token.json`, source of
truth in Bitwarden. OAuth app "Hermes-Agent" is in production (tokens do not expire weekly).
Re-auth: `~/.hermes/scripts/google_auth_queue.sh` on the VM + SSH tunnel on port 53682 from the PC;
**Filip clicks the consent himself**. Sending mail or deleting files needs his OK.

## Backups

`~/.hermes/scripts/hermes_backup.sh` (run now) · `list` · `restore <account> <file> [dir]` ·
`archive` (local, unencrypted, for tests). Timer `hermes-backup.timer` 03:00 Europe/Warsaw,
`Persistent=true`. Retention: see `WOBI/ansible/roles/hermes_backup/README.md` (the WOBI version does not
delete old archives; the older skill said 7 kept). Without the two rclone crypt passwords in Bitwarden the backups cannot be decrypted.
Dev backs up its work to `praktyki:hermes-dev-backups-encrypted`.

## Notion

- "Hermes Organized" (root of Hermes knowledge), database **"Baza wiedzy Hermesa"** (properties: Źródło,
  Typ, Kategoria, Projekt, Profil, Przegląd, Wartość, Tagi), "Przegląd wiedzy: …" overview pages,
  "🧭 Hermes — Plan dnia" (timeplane), "Przegląd sesji Hermesa" (quality review of all sessions),
  "Funkcje Hermesa" (mirror of the feature registry), "Hermes — kontekst dla agentów" (mirror of this repo).
