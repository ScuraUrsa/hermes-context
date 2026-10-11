# Hermes on three VMs: prod, stage, dev (proposal, 2026-10-09)

Filip's ask (2026-10-09 10:45): production for daily use and conversation, a stage that only updates and
migrates, and a dev workspace where the code work happens. Changes go dev → stage → prod. Prod syncs as
little as possible and needs fewer resources. Prod may ask Filip to test something on dev or on stage.

## 1. What runs where

| | **prod** (always on) | **dev** (always on, or on while working) | **stage** (on demand) |
|---|---|---|---|
| Purpose | talk to Filip, act on his behalf | code: repos, worktrees, kanban workers | candidate of the next prod |
| Hermes | gateway + chat platforms, cron (timeplane, quota, backups), memory | Hermes as **worker only**: kanban dispatch, coder/reviewer profiles, Claude Code, gm_run; **no chat platforms** (no second bot) | full prod setup built from the candidate refs, **no** chat platforms, no uploads |
| Kanban board | client (proxy, see §2) | **owner** of `kanban.db` and every workspace | test copy |
| Repos, worktrees, `hermes-sandbox`, `k8s_research`, … | none | all of them | none (built clean) |
| Honcho (long-term memory) | yes (source of truth) | reads prod's Honcho API (read-only, host-only network) | restored copy, for migration tests |
| Home Assistant, whisper, piper, notification bridge, timeplane intake | yes | no | yes (for tests, its own HA URL) |
| Syncthing (Obsidian) | yes | no | no |
| Tailscale | node `hermes-vm-1` / 100.66.185.39 (phone, HA app) | own node `hermes-dev` (Termux `c` = Claude Code) | own **ephemeral** node `hermes-stage`, so Filip can test from the phone |
| CrewAI, docker builds, tofu/opa/hubble, CI tools | no | yes (CrewAI only if turned on) | no |
| Backup | state only: `~/.hermes`, Honcho, HA, Syncthing identity, identities (small, fast) | work: branches pushed to GitHub + `work.tar` of dirty worktrees and kanban.db | none (rebuilt every time) |
| RAM (measured today) | ~1.7 GB idle, ~3.5 GB with embeddings → **4–5 GB VM** | **6 GB** (builds, workers) | **5–6 GB**, only during an upgrade |
| IP (host-only) | 192.168.56.107 | 192.168.56.108 | 192.168.56.110 |

The laptop has 32 GB; Windows and Chrome use ~22 GB right now, so prod + dev (≈11 GB) fit always and stage
fits while it exists if dev is paused (the daemon can save dev's state and restore it afterwards).

## 2. The proxy: prod talks, dev works

Only one thing crosses from prod to dev: **work orders**. Everything else stays local.

```
Filip ── chat ──► prod Hermes ──(kanban API over host-only net, token)──► dev: kanban board + workers
                     ▲                                                    │
                     └──── task status, results, PR links, "please test" ◄┘ (dev pushes events)
```

* **Kanban moves to dev.** Prod gets a thin `kanban` proxy: `hermes kanban create/list/show/comment` on
  prod call dev's board (Hermes API server with `HERMES_API_SERVER_KEY`, or `ssh dev hermes kanban …` as a
  first step). Workspaces like `worktree:/home/ubuntu/crewai-orchestrator` resolve on dev, where the repos are.
* **Prod scripts that read `kanban.db` today** (timeplane, quota_governor, planner_context, active_terminal,
  gh_actions_watchdog) read a **read-only snapshot** that dev pushes to prod every minute (a few MB), or use
  the proxy. No other data goes from dev to prod.
* **Dev → prod events**: when a task finishes, needs a decision or wants a test, dev posts to prod's intake
  (like timeplane's), and prod tells Filip in chat ("PR #12 is ready, please test X on dev/stage").
* **Memory**: dev workers get the context they need in the task body; for more, they query prod's Honcho
  read-only. Dev never writes Filip's memory.
* **Secrets**: each VM reads its own from Bitwarden with its own machine token and only the keys its role needs
  (dev: GitHub, Ollama, Claude; prod: chat platforms, Google, HA, bridge). Nothing is copied between VMs.

## 3. Flow of changes: dev → stage → prod

1. Code changes happen on dev (Hermes workers or Filip with Claude Code in Termux), as branches + PRs in
   GitHub (WOBI, hermes-agent fork, hermes-timeplane, …).
2. A change meant for prod is merged into the **`stage` branch** of the repos it touches (WOBI pins point
   to the new commits/versions: Hermes, HA image, Honcho, …).
3. The host daemon builds **stage from zero** from the `stage` refs (what we did today), restores prod's
   latest backup into it, runs the migration and the compatibility tests, and keeps it running.
4. Prod asks Filip: "stage has X, please try it" (HA on stage, `hermes-stage` chat in Termux), or for a dev-only
   check: "try branch Y on dev". Filip answers in chat.
5. Approved → **cutover stage → prod** (today's mechanism: pause, final backup, identity), `stage` merged to
   `master`. The old prod stays off for 24 h for rollback, then the stage slot is free again.
6. Hermes' own upgrades (new Hermes release, new HA, …) take the same path; small ones can be automatic later
   (policies).

## 4. Order of work

1. **Inventory and roles in WOBI**: hosts `prod`, `dev`, `stage`; role toggles per host (prod: no workspaces/
   CrewAI/build tools; dev: no gateway platforms/HA/Syncthing; stage: like prod, no uploads, ephemeral
   Tailscale). Hermes "worker mode" for dev (gateway off, kanban dispatcher on).
2. **Build dev** from zero; move repos, worktrees and `kanban.db` from prod to dev with `hermes_work.py`
   (the same pack/unpack the cutover used) at a quiet moment with kanban paused.
3. **Proxy on prod**: kanban client to dev, kanban.db snapshot for timeplane & co, dev → prod event intake.
4. **Slim prod**: remove repos/worktrees/build tools, RAM 8 → 5 GB, backup without work.
5. **Stage pipeline** in the daemon: build from `stage` refs, test suite, "please test" message, cutover.
6. Rename VMs to `prod`, `stage`, `dev` (VirtualBox, hostnames, WOBI inventory) after tomorrow's finalize.

## 5. Decisions for Filip

* **A. Where the kanban board lives**: on dev (recommended: work and its data in one place) or on prod.
* **B. Tailscale for dev and stage**: own nodes `hermes-dev` and ephemeral `hermes-stage` (recommended; needs
  one tagged, reusable auth key in Bitwarden) or host-only only (no phone access to dev/stage).
* **C. Dev always on** (recommended while Hermes works in the background) or only when there is work.
