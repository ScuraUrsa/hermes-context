# 6. Conventions

## Git and delivery

- **Every change goes through a pull request.** Until 2026-10-08 Hermes pushed straight to `master`/`main`;
  since 2026-10-08 the *delivery gate* requires branch → PR → a separate review task (reviewer profile or
  Filip) → merge. Agents from other projects follow the same rule: open a PR, never push to a default branch.
- Branch names: Hermes kanban workers use `wt/t_<task-id>` (one git worktree per task); Claude sessions
  use `claude/<topic>`; IaC work in WOBI used `iac/<task-id>`; upgrades `upgrade/<app>-<version>`.
  Profile workspace repos document `feature/*`, `fix/*`, `chore/*`, `spike/*`.
- Delivery gate implementation: `crewai-orchestrator/scripts/delivery_gate.py` + `pr_merge_watch.py`, runbook
  `docs/RUNBOOK-delivery-gate-pr-review-tasks.md`. A finished task = pushed branch + PR + review task (Notion
  or Google Calendar for Filip). `master`/`main` merges are Filip's; low-risk auto-merge was designed for a
  `hermes-operational` branch that does not exist yet.
- Commits use conventional prefixes (`feat:`, `fix:`, `ci:`, `docs:`, `chore:`), often with `(t_<id>)`;
  PRs are squash-merged.
- PR titles from Hermes carry a risk tag: `[low risk]` / `[high risk]`. High risk (architecture, secrets,
  networking, Terraform state, cross-repo, breaking changes) needs Filip
  (`hermes-infra/AUTO_MERGE_POLICY.md`).
- Default branches differ per repo (`main`, `master`, `aks_private`, `ai_stuff_save`); check before branching.
- Stage paths explicitly in shared checkouts (`git add <paths>`, never `git add -A`); several agents work
  in `D:\hermes` and on the VMs at the same time.
- Don't rewrite history on shared branches; merge the base in instead.

## Decision logs

- Every non-trivial task leaves a decision log in the repo it changed:
  `docs/decision_log_<topic>_t_<task-id>.md` (crewai-orchestrator) or `docs/DECISION_LOG_<topic>.md` (WOBI),
  or a root `decision_log.md` in small repos. Contents: goal, what was checked, decision, alternatives,
  evidence (commands, commits), follow-ups.
- `crewai-orchestrator/docs/DECISION-LOG-INDEX.md` indexes them. Append-only; do not bump shared counters
  in parallel PRs (it caused red master ~30 times).
- Plans and designs for Filip are written as Markdown files (in `tasks/` on the PC, project files, or the
  repo's `docs/`) and mirrored to Notion when he needs to read them.

## Code and CI

- Python: ruff (pinned), pytest; tests must run offline (no live services, no real installs in unit tests).
- Ansible: ansible-core 2.17, ansible-lint production profile, idempotent roles, FQCN, `no_log` on secrets.
- Shell: shellcheck. Pin third-party versions (image digests, tool versions with sha256).
- Forked upstream workflows (hermes-agent): guard fork-only breakage with
  `if: github.repository == 'NousResearch/hermes-agent'` rather than deleting upstream files.
- Scheduled workflows must fail loudly (no `2>/dev/null || echo`); avoid creating a new branch per run.

## Language and style

- Talk to Filip in Polish, short and concrete; code, commits and most docs in English.
- Notion pages for Filip in Polish.
- Reports state what was checked (command, file:line, commit) and what was inferred.

## Safety rules for agents

- Read-only on prod unless the task explicitly says otherwise; experiment on dev, a replica (via the host
  daemon) or stage.
- Never print or commit secret values; reference Bitwarden key names. Redact remote URLs.
- Never mint tokens, grant OAuth, send mail, delete user files, or spend money without Filip's explicit OK.
- Don't start the CrewAI stack (retired 2026-10-09).
- Don't change Filip's other calendars (read-only); timeplane only writes to "iZadania".
- A VM may not delete itself; deleting a VM you didn't create needs approval.
