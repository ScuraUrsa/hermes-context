# 3. Repository map (ScuraUrsa, state 2026-10-10)

Legend: **core** = needed to run Hermes today · **active** = being worked on · **dormant** = no recent work,
keep for reference · **legacy** = superseded · ⚠️ = contains leaked secrets in files or history (never copy).

## Core: run and deploy Hermes

| Repo | Default branch | Role | Status |
|---|---|---|---|
| [WOBI](https://github.com/ScuraUrsa/WOBI) | `master` (newest work on **`iac/final`**, ~130 commits ahead) | **Source of truth for VMs**: Ansible roles (`ansible/`), `pull.sh`, versioned `~/.hermes/scripts`, `features/registry.yaml`; old CrewAI app copy | core, active |
| [hermes-infra](https://github.com/ScuraUrsa/hermes-infra) ⚠️ history | `main` | `hermes-host-daemon` (Windows, VM lifecycle, replicas, cutover, approvals), `phone-control/`, docs (`self-upgrade-flow.md`), Coder SOUL.md, config snapshots | core, active |
| [hermes-agent](https://github.com/ScuraUrsa/hermes-agent) | `main` | Fork of NousResearch Hermes Agent: provider semaphore, kanban goal-loop, vm-lifecycle skill, CI fixes; PR #5 merges upstream Jun→Oct. **VMs run upstream v0.20.0 (a6ede70), not this fork** | core, active |
| hermes-timeplane (private, not in this session's scope) | – | Time management: `tp` CLI, plugin, skill, intake service, HA packages; deployed by WOBI role `timeplane` | core, active |
| [crewai-orchestrator](https://github.com/ScuraUrsa/crewai-orchestrator) | `master` | Old CrewAI app (retired) + **work log of the autonomy loop**: ~900 docs, decision logs, runbooks (`RUNBOOK-autonomy-loop.md`, delivery gate), YouTube archive scripts, `delivery_gate.py` | active (docs/scripts), app retired |
| [packer-images](https://github.com/ScuraUrsa/packer-images) | `master` | Optional `hermes-base` VirtualBox image (Ubuntu 22.04 + WOBI base/docker) | core-optional |
| [terraforge-stacks](https://github.com/ScuraUrsa/terraforge-stacks) | `aks_private` (local VM work on `glados`, `iac/final`) | Terraform + `manage-local-vm.ps1` for VirtualBox VMs; Azure AKS stacks (2025) | core-optional / Azure dormant |
| [hermes-notification-bridge](https://github.com/ScuraUrsa/hermes-notification-bridge) | `master` | aiohttp relay :8655, push_message tool, WebSocket/SSE clients | prod-experimental, 0 clients |

## Add-ons and experiments

| Repo | Default branch | What | Status |
|---|---|---|---|
| [hermes-enhancements](https://github.com/ScuraUrsa/hermes-enhancements) | `master` | Research + POVs (July); now home of new tools: hbench (#6), vbench (#7), video-digest (#8) drafts, Zed eval-cli release | active (draft PRs) |
| [hermes-gui-automation](https://github.com/ScuraUrsa/hermes-gui-automation) | `master` | `hermes_gui` library (AT-SPI2, OCR, VLM, Playwright), skill, Ansible; PR #2 `hermes-teach` (record user actions → skills) | revived 2026-10-10 |
| [hermes-voice-assistant](https://github.com/ScuraUrsa/hermes-voice-assistant) | `master` | Voice stack design (Open WebUI voice, HA Wyoming, wake word); real voice now runs via WOBI `home_assistant` | dormant |
| [hermes-conduit-bridge](https://github.com/ScuraUrsa/hermes-conduit-bridge) | `master` | Flutter/Android client stub for the bridge; APK never built | dormant |
| [hermes-contact-crm](https://github.com/ScuraUrsa/hermes-contact-crm) | `main` | Contact CRM in Notion + parsers (FB, WhatsApp, Instagram, vCard) | dormant |
| [hermes-trade](https://github.com/ScuraUrsa/hermes-trade) | `main` | Algorithmic trading scaffold (FinBERT, brokers) v0.1.0 | dormant |
| [hermes-office-suite](https://github.com/ScuraUrsa/hermes-office-suite) | `master` | Empty seed | empty |
| [nextcloud-deployment](https://github.com/ScuraUrsa/nextcloud-deployment) | `master` | Nextcloud for 50 users design + Ansible (business idea) | dormant, CI green |
| [ra-pipeline](https://github.com/ScuraUrsa/ra-pipeline) | `master` | Research agent: daily brief / weekly ideas to Notion | dormant |
| [hermes-agent-self-evolution](https://github.com/ScuraUrsa/hermes-agent-self-evolution) | `main` | Unmodified upstream DSPy/GEPA self-evolution | reference |
| [agentskills-private](https://github.com/ScuraUrsa/agentskills-private) | `jh/add-best-practices-guide` | Clone of the agentskills.io spec | reference |
| aethershell (not in this session's scope) | – | Rust project with CI watched by the CI doctor | unknown |

## Profile workspaces (one per Hermes profile, created 2026-06-01)

`hermes-architect-workspace`, `hermes-coder-workspace` (CrewAI healthcheck), `hermes-default-workspace`
(copies of patched hermes-agent files, decision log, YouTube status), `hermes-maintainer-workspace`,
`hermes-researcher-workspace`, `hermes-reviewer-workspace`, `hermes-writer-workspace`,
`hermes-copilot-pro-workspace`, `hermes-copilot-archive-workspace`. Mostly README only; workers now work
in worktrees of the real target repos instead.

## Legacy

| Repo | Why legacy |
|---|---|
| [hermes-config-backup](https://github.com/ScuraUrsa/hermes-config-backup) ⚠️ | Raw `~/.hermes` snapshot from July 2026 with secrets in plain text; superseded by the encrypted backup |
| [terraforge-modules](https://github.com/ScuraUrsa/terraforge-modules) ⚠️ | Azure modules; tracks an SSH private key |
| [terraforge-pipeline](https://github.com/ScuraUrsa/terraforge-pipeline) | Azure DevOps-era Ansible/terraspace; harvested into WOBI |

## Outside GitHub

- `D:\hermes` on Filip's PC: working clones, `HERMES_MAP.md` (changelog + broken-things list),
  `tasks/*.md` plans, `_iac/`, `_timeplane_work`, `terraforge-stacks/local/generated` (VM SSH keys).
- Notion "Hermes Organized" tree (see [05-operations.md](05-operations.md#notion)).
- Project files of the Claude project "Hermes_local" (designs, session review raw data).
