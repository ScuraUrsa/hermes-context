# 1. What Hermes is for Filip

## In one paragraph

Hermes is Filip's personal, always-on AI assistant. It is the open-source
[Hermes Agent](https://github.com/NousResearch/hermes-agent) by Nous Research (the VMs run upstream v0.20.0, pinned in WOBI;
Filip's fork `ScuraUrsa/hermes-agent` carries patches and an upstream-merge PR) inside Ubuntu VMs on his Windows laptop (VirtualBox). The long-term goal of
the project is stated as: **"a 24/7 local Hermes that replaces using the phone and the PC; it should be
able to do everything that can be done virtually."** Filip talks to it (Polish by default) from the
PC terminal, from the phone (Termux over Tailscale, Home Assistant app with voice), and it works on its own
in the background (cron jobs, a kanban board of tasks, coding through Claude Code).

## Who Filip is, as far as Hermes needs to know

- Speaks and writes Polish; most repo docs and code comments are in English. Answer him in Polish.
- DevOps / cloud background (Azure, Terraform, Ansible, Kubernetes). Many repos date from that work.
- Has a child; family time matters (timeplane rules such as "after 20:00 family only").
- Uses 4 Google accounts (aliases `fjkazmierczak` (default), `kazmierczakfilipjan`, `koalamis6`,
  `praktyki`), Notion as the main knowledge base, Obsidian (synced to the VM), Bitwarden for secrets,
  a Samsung S20 FE phone.
- Pays for: Claude subscription (used only through the official Claude Code CLI), two Ollama Cloud Pro
  plans, Notion. No Anthropic API keys (his rule).

## What Hermes is used for today (use cases)

| Area | What Hermes does | Where it lives | Status (2026-10-10) |
|---|---|---|---|
| Conversation and daily help | Chat in the terminal (`hermes`), answers, research, plans (e.g. a Bergamo/Como/Zadar trip plan) | prod VM, `hermes` CLI | working |
| Time management ("timeplane") | Plans Filip's day from his calendars, manages Google Tasks list "iZadania", breaks, moods, rules in natural language, evening review; Notion "🧭 Hermes — Plan dnia" | `ScuraUrsa/hermes-timeplane`, WOBI role `timeplane` | prod, experimental (review gate 2026-10-15) |
| Voice | Home Assistant Assist pipeline "Hermes (pl)": Whisper (pl) + Hermes + Piper (pl_PL gosia) | WOBI role `home_assistant` | prod, experimental |
| Phone control | Hermes and Claude can drive the Android phone (adb tunnel via the PC, `phone` CLI, skills) | `hermes-infra/phone-control`, WOBI role `phone_control` | merged, early |
| Notifications to the phone | push_message → notification bridge (:8655) | `hermes-notification-bridge`, WOBI role | prod, experimental, 0 real clients; Matrix proposed instead |
| Long-term memory | Honcho (self-hosted) shared by all profiles; built-in MEMORY.md / USER.md | WOBI role `honcho` | prod, stable |
| Notes | Obsidian vault synced laptop ↔ VM with Syncthing | WOBI role `syncthing` | prod, stable |
| Google | Gmail, Drive, Calendar, Tasks, YouTube for all 4 accounts (`gacct`, rclone) | `~/.hermes/scripts`, skills | working |
| Autonomous work 24/7 | Planner creates tasks from a roadmap, workers (profiles) execute them on the kanban board, Claude Code writes the code, results go to GitHub through PRs | dev VM, `crewai-orchestrator` (docs, scripts) | working, being reviewed (quality/cost) |
| CI care | `gh-actions-watchdog` / "CI doctor" opens fix tasks for red GitHub Actions in ScuraUrsa repos | WOBI scripts | dev |
| YouTube | Archive of subscriptions/playlists, transcripts; "video-digest" (watch videos for Filip, write Notion report) in progress | `crewai-orchestrator/scripts/youtube_*`, `hermes-enhancements` PR #8 | partly built |
| Contacts | Personal CRM in Notion: `contacts show Mama`, `contacts met Buła` (aliases, last contact, notes) | `hermes-contact-crm` (private, PR #1), CLI `contacts`, skill `contact-crm` | live on prod since 2026-10-11, not in WOBI yet |
| Private chat (planned) | Matrix server (Continuwuity) on the tailnet for chat with Hermes and alerts to a locked phone | `~/matrix` on prod | server up since 2026-10-09; bot not created; not in WOBI or the backup |
| Knowledge base | Writes findings to Notion ("Baza wiedzy Hermesa" database) | Notion | working |
| Self-maintenance | Nightly encrypted backup, rebuild/cutover of its own VM, upgrades through dev → stage → prod | WOBI, hermes-infra | prod-experimental |

Older experiments (June–August 2026), mostly dormant: GUI automation, office suite, contact CRM in Notion,
voice assistant skeleton, trading system, Nextcloud deployment design, research pipeline, Conduit bridge
(Android client). See [03-repo-map.md](03-repo-map.md).

## Filip's standing rules (do not break)

1. **Nothing public.** Services listen on localhost, the host-only network or Tailscale only.
2. **No secrets in Git, logs, PR text, cloud-init or chat.** Secrets live in Bitwarden Secrets Manager and
   are read at run time. (Several old repos *do* contain leaked secrets; never copy from them.)
3. **Coding = Claude Code CLI** on Filip's subscription (`cc_run.py`), never Anthropic API keys, never a
   Claude token inside Hermes' own `anthropic` provider, never the env name `CLAUDE_CODE_OAUTH_TOKEN`.
   Thinking / non-coding = Ollama Cloud models.
4. **Filip clicks OAuth consents himself.** Agents do not grant themselves OAuth access, do not create
   tokens on his behalf (an HA admin token Hermes once minted itself is considered a mistake).
5. **Sending mail, deleting files, posting anything outside the project, spending money** needs his
   explicit OK.
   Note: Hermes itself runs with `approvals.mode: off` (no confirmation prompts), so these rules live in
   skills, SOULs and memory, not in a technical gate.
6. **Changes go through PRs with review** (since 2026-10-08). No direct pushes to default branches.
7. **The live VM is not a lab.** Experiments happen on dev, a replica, or stage; prod gets tested changes.
8. Filip's own tasks always beat Hermes' self-generated work (priority 100 vs 0–10), and self-work has a
   hard USD budget.
