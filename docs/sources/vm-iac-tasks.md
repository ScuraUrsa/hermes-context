# Hermes VM as code: analysis and subagent task list

Date: 2026-10-07. Asked by Filip: read WOBI, crewai-orchestrator, terraforge-* and packer-images,
then list the tasks (for subagents) that make the VM fully deployable from these repos,
with every missing piece of configuration handled by Terraform, Ansible or similar tooling.

Read with: `HERMES_MAP.md`, `hermes-infra/docs/self-upgrade-flow.md` (branch `vm-replica-upgrade`).

---

## 1. Findings in short

1. **Nothing can rebuild today's unity-orchestrator.** It was created on 2026-07-23 with
   terraforge-stacks + WOBI, then changed by hand for 2.5 months. The first rebuilt test copy
   (2026-10-07, `hermes-replica-db729388`) got Hermes, its state and Honcho back. It lacks Home Assistant,
   wyoming, the notification bridge, Syncthing, Tailscale, Ollama models, launchers, and Honcho's real config.
2. **Provisioning code exists four times and the copies disagree.** WOBI `ansible/` (branch
   `mastersynchro` only), terraforge-pipeline `remote_command_executor/recipes/*hermes*` (branch `main`),
   packer-images `orchestrator/playbook.yaml` (+ a copy in terraforge-stacks `main:local/packer`), and
   the replica bootstrap in hermes-infra. None of them describes the current VM.
3. **Branches are split.** WOBI: `master` (default, 2026-08-28, CrewAI code and tests by Hermes) vs
   `mastersynchro` (2026-07-23, the only branch with the Ansible playbooks used to build the VM). The VM's
   `~/WOBI` is 1 commit ahead of `origin/master` (unpushed). terraforge-stacks/modules: the local VirtualBox
   work is on `glados`; the defaults are the 2025 Azure branch `aks_private`; `main` has a packer copy.
4. **CrewAI code exists twice:** WOBI `crewai/ docker/` and crewai-orchestrator `crewai/ docker/`.
   crewai-orchestrator is also full of POCs that Hermes' Kanban workers added (backstage, consul, crossplane,
   argocd, loki, opentofu …) and hundreds of `wt/t_*` branches. The VM runs the CrewAI compose from
   `~/crewai-orchestrator/docker`. Filip stopped that stack on 2026-10-06 ("don't start").
5. **The host-only IPs are not what they seem.** The VirtualBox DHCP server hands out 192.168.56.101-254.
   unity-orchestrator gets .107 **by DHCP** (netplan `dhcp4: true`). The daemon's static range for replicas
   (.110-.199) overlaps that pool, so an IP clash is possible.
6. **Secrets sit in Git.** terraforge-modules tracks `ssh/id_rsa`, hermes-infra tracks `.env`
   (Bitwarden bootstrap token), and 8 repos on the VM have PATs in their remote URLs. WOBI still has an
   Azure Key Vault path; Bitwarden is the real source now.
7. **packer-images CI never ran:** the workflow needs a `[self-hosted, windows, virtualbox]` runner
   that does not exist (runs cancelled or failed since July).

## 2. Repositories: role today and verdict

| Repo | Today | Verdict |
|---|---|---|
| **WOBI** | Ansible playbooks/tasks (only on `mastersynchro`), CrewAI app copy, docker compose, old hermes backup scripts, cloudflare tunnel module | **Canonical provisioning repo** (HERMES_MAP already calls it the single source of truth). Merge branches, restructure into roles, drop the CrewAI copy. |
| **terraforge-stacks** | `local/`: Terraform + `manage-local-vm.ps1` that built unity-orchestrator (branch `glados`); Azure stack on other branches | **Terraform entry point for local VMs.** Make `glados` work the default for local, declare the Hermes VM(s) and IP reservations. |
| **terraforge-modules** | Azure modules (+ unused `local_vm`), `ssh/id_rsa` tracked | Azure only, dormant. Remove the key. |
| **terraforge-pipeline** | Azure DevOps-era Ansible roles; `main` has 6 Hermes "recipes" | Legacy. Harvest anything useful from the Hermes recipes into WOBI, then mark the repo dormant. |
| **packer-images** | Ubuntu 22.04 autoinstall OVA + playbook; CI needs a missing self-hosted runner | **Optional base image.** The cloud image works today. Keep Packer for baking slow, stable layers later (Docker, base packages); build locally, fix or remove the CI. |
| **crewai-orchestrator** | CrewAI app + compose that the VM runs, plus many POCs and Kanban branches | **CrewAI app only.** Provisioning uses its compose as an optional role, off by default. Archive the POCs. |
| **hermes-infra** | host daemon (replicas, approved cutover) + replica bootstrap | Daemon stays here. The bespoke bootstrap is replaced by a call to WOBI's playbook. |

## 3. Live VM vs repos: what is missing

Checked read-only on unity-orchestrator and compared with the fresh replica (2026-10-07).

| # | Configuration on the live VM | In any repo? | Target |
|---|---|---|---|
| 1 | Hermes code (upstream v0.20, `~/.hermes/hermes-agent`, venv) | installer only | Ansible role `hermes` (pinned repo/ref) |
| 2 | Hermes state (config, memories, skills, profiles, cron, DBs) | nightly encrypted backup | `hermes_state` role: restore from backup (it is data, not config) |
| 3 | Newer-Hermes config keys (`secrets.bitwarden.server_url`, honcho plugin) | replica bootstrap only | `hermes` role |
| 4 | `hermes-gateway.service` + drop-ins (CLAUDE_CONFIG_DIR, ollama_keys ExecStartPre) | hermes-infra/replica/units | `hermes` role |
| 5 | `hermes-backup.service/timer`, `hermes-curator.service/timer` | partly (units), scripts only in backup | `hermes_backup` role (scripts into a repo) |
| 6 | Custom `~/.local/bin/hermes` launcher + profile launchers (architect, coder, …), `gacct` | no | `hermes` role (files in repo) |
| 7 | Claude Code CLI 2.1.x in `~/.local/bin/claude`, `CLAUDE_CONFIG_DIR=~/.claude-code`, `cc_run.py` | no (state in backup) | `claude_code` role |
| 8 | Honcho: upstream clone + **gitignored** `docker-compose.yml` (telemetry-sink, extra_hosts), `config.toml` (deepseek-v4.1-flash via Ollama Cloud, local embeddings), `ollama/Modelfile.honcho-embed`, `telemetry/sink.py`, `.env` | no (only `.env` in backup) | `honcho` role with templates; secrets from Bitwarden |
| 9 | Ollama (systemd) + override (`OLLAMA_HOST=0.0.0.0`, `OLLAMA_MODELS=/opt/models`, `LLAMA_ARG_CACHE_RAM=0`) + models `honcho-embed:qwen3-0.6b-ctx512`, `qwen3-embedding:0.6b`, `tinyllama:1.1b` | WOBI installs Ollama only | `ollama` role (override + model list + Modelfile) |
| 10 | Home Assistant (`~/homeassistant/docker-compose.yml`, `config/configuration.yaml`, custom component `openai_compatible` 0.2.0) | no | `home_assistant` role; HA runtime data restored from backup (add to backup) |
| 11 | wyoming-whisper (small-int8, pl) + wyoming-piper (pl_PL-gosia-medium) in the HA compose | no | `home_assistant` role |
| 12 | hermes-notification-bridge (image built from `~/hermes-notification-bridge`, `docker run`, volume `bridge-data`, :8655) | repo yes, deploy no | `notification_bridge` role (compose file) |
| 13 | Syncthing (apt repo, `syncthing@ubuntu`, folder `obsidian-filip` ↔ laptop, GUI password in BWS) | setup script in `D:\hermes\obsidian-setup` | `syncthing` role. A new VM gets a new device ID, so pairing needs Filip once (or keep the old device keys via backup) |
| 14 | Tailscale (apt repo, hostname `hermes-vm`, `--ssh`, auth key in BWS) | no | `tailscale` role; cutover handles logout/rename |
| 15 | Docker CE from docker.com (replica got Ubuntu `docker.io`) | WOBI `install_docker` recipe | `docker` role (CE) |
| 16 | apt: gh, mosh, openjdk-17, ffmpeg, poppler-utils, rsync, ripgrep, dev headers; repos docker/github-cli/syncthing/tailscale | partly | `base` role package list |
| 17 | `/usr/local/bin`: bws, ansible, packer, kubectl, helm, kind, flux, cilium, consul, nomad, vault; `~/.local/bin`: tofu, opa, hubble | no | `base` role (bws, ansible); POC tools: optional `devtools` role or drop |
| 18 | rclone 1.75 + `~/.config/rclone/rclone.conf` (Google remotes) | conf in backup | `base` installs rclone; conf restored from backup or rendered from BWS |
| 19 | CrewAI stack (compose project `docker` from crewai-orchestrator; postgres published on 0.0.0.0:5432, litellm on :4000) | WOBI + crewai-orchestrator | `crewai` role, **off by default**; bind to localhost when on |
| 20 | Network: host-only .107 by VirtualBox DHCP, NAT on enp0s8 | terraform passes a static IP the VM does not use | Terraform: DHCP reservation per VM (or static netplan everywhere) |
| 21 | VM size: 4 vCPU, 8 GB RAM (since 2026-10-07), 80 GB disk; autostart task on Windows | Terraform defaults differ (8 GB/80 GB, "minimal" CrewAI mode) | terraforge-stacks `hermes.tfvars` |
| 22 | Leftovers: 7 exited `http-echo` containers (nomad/consul POC), unused volumes | - | cleanup task (no IaC) |
| 23 | Git remotes with PATs, `~/.git-credentials` | - | `base` role: Bitwarden credential helper (as in the replica bootstrap) |

## 4. Target design

```
packer-images (optional)  ──► base OVA (or the Ubuntu cloud image, as today)
terraforge-stacks/local   ──► VirtualBox VM(s): size, NICs, DHCP reservation, cloud-init seed (SSH key only)
hermes-host-daemon        ──► same VM creation for replicas (shares manage-local-vm.ps1)
WOBI ansible (site.yml)   ──► everything inside the VM, by role:
     base · docker · bws · tailscale · syncthing · ollama · honcho · hermes · claude_code
     hermes_backup · home_assistant · notification_bridge · crewai (off) · role_standby/active
Nightly backup            ──► Hermes data only (state.db, kanban.db, memories, HA data, Honcho DB …)
Bitwarden                 ──► every secret; nothing secret in Git or cloud-init
```

- Ansible runs **on the VM itself** (`ansible-pull` of WOBI), because the Windows host has no native Ansible.
  Terraform's existing WSL step and the daemon both just start `ansible-pull` over SSH.
- One playbook for both cases: a fresh `unity-orchestrator` and a replica (`hermes_role: standby`).
- Host variables for `unity-orchestrator` live in WOBI `inventory/host_vars/`. Secrets are looked up from
  Bitwarden at run time (the `bws` CLI, or the `community.general.bitwarden_secrets_manager` lookup).

## 5. Tasks for subagents

**Rules for every subagent** (repeat in each prompt):
- Work on a branch `iac/<task-id>` in the named repo; open a draft PR; never push to the default branch.
- No secrets in Git, logs, PR text or cloud-init. Read them from Bitwarden on the VM (`bws`, token in
  `~/.bws_token`) and pipe them straight to their target. Never print `git remote -v` unredacted.
- The live VM (`ubuntu@192.168.56.107`, key `D:\hermes\terraforge-stacks\local\generated\id_rsa`) is
  **read-only**. Test only on the replica (`192.168.56.110`) or a sandbox VM made through the host daemon.
- Done means: `ansible-lint` / `terraform validate` pass, the role ran twice on the replica with no changes on
  the second run (idempotent), and the PR lists what was verified.

### Wave 1: foundations (parallel)

| ID | Repo | Task | Acceptance |
|---|---|---|---|
| **T1** | WOBI | Merge `mastersynchro` into `master` (keep Hermes' tests and CrewAI changes from `master`); push the VM's unpushed commit first. Restructure `ansible/` into `roles/`, `site.yml`, `inventory/{hosts.yml,host_vars/unity-orchestrator.yml,group_vars}`, `requirements.yml`; move CrewAI app code out (it lives in crewai-orchestrator). Add CI: `ansible-lint` + syntax check. | One branch with the roles skeleton; CI green; old playbooks still callable or deleted with a note. |
| **T2** | WOBI | **Capture live config (read-only)** into role files/templates, items 6, 8-14, 16-17 of the table: Honcho compose/config.toml/Modelfile/telemetry sink, Ollama override + model list, HA compose + `configuration.yaml` + component pin, wyoming settings, notification bridge run config, Syncthing folder config, Tailscale prefs, launchers, unit files, apt repos and packages. Replace every secret with a Bitwarden lookup name. | Files under `roles/*/files|templates`; `grep` finds no secret values; a table maps each captured item to its source path on the VM. |
| **T3** | terraforge-stacks | On `glados`: add a `hermes` deployment mode (one VM `unity-orchestrator`: 4 vCPU, 8 GB, 80 GB), drop the CrewAI-centric "minimal/full" defaults from that path, set a **DHCP reservation** per VM (`VBoxManage dhcpserver modify --vm … --fixed-address`), output an Ansible inventory, and replace the WSL Ansible step with `ansible-pull` over SSH. Delete the duplicate `local/packer` from `main` (Packer lives in packer-images). Make `glados` the default branch or merge it into `main` (ask Filip). | `terraform validate`; a dry plan shows the VM matching the live one; README updated. |
| **T4** | packer-images | Decide CI: remove the workflow, or document how to register the laptop as a self-hosted runner. Make the image optional: align its baked packages with WOBI's `base` + `docker` roles and output the OVF path Terraform and the daemon use. | `packer validate` passes; README says how and when to rebuild. |
| **T5** | several | **Security cleanup** (needs Filip for rotation): remove `ssh/id_rsa` from terraforge-modules and `.env` from hermes-infra (history rewrite only if Filip agrees), switch the VM's repos from PAT URLs to the Bitwarden credential helper, list what must be rotated. | No secret file in the trees; VM remotes have no credentials; rotation list delivered to Filip. |

### Wave 2: roles (parallel after T1, T2)

| ID | Role(s) | Notes |
|---|---|---|
| **T6** | `base` + `docker` + `bws` | Packages and apt repos (item 16), timezone, a swap file (the VM has none, 8 GB RAM), Docker CE, bws CLI pinned, git credential helper, rclone. |
| **T7** | `hermes` + `claude_code` | Pinned Hermes repo/ref (the live VM tracks upstream, v0.20); config migrations (`server_url`, honcho plugin); gateway unit + drop-ins; launchers; `hermes_role: active/standby` (port `hermes-role` from hermes-infra); Claude Code CLI + `CLAUDE_CONFIG_DIR`. |
| **T8** | `hermes_state` + `hermes_backup` | Restore the newest encrypted backup (port the logic from `hermes-host-daemon/replica/hermes-bootstrap.sh`); install backup and curator units; move `hermes_backup.sh`, `ollama_keys.py`, `cc_run.py` and the other `~/.hermes/scripts` into a repo so they are not only in the backup. Add HA data and Honcho config to the backup. |
| **T9** | `ollama` + `honcho` | Ollama override and models incl. the `honcho-embed` Modelfile; Honcho clone pinned to a commit, templated compose/config.toml/.env, telemetry sink, DB restore from backup. |
| **T10** | `home_assistant` | Compose (HA + wyoming whisper/piper), `configuration.yaml`, `openai_compatible` component pinned, Hermes API connection from Bitwarden; HA state from backup. |
| **T11** | `notification_bridge` + `syncthing` + `tailscale` | Bridge as a compose service built from its repo; Syncthing service + folder (pairing step documented); Tailscale join with auth key from Bitwarden, hostname `hermes-vm`, SSH on. |
| **T12** | `crewai` (optional) | Compose from crewai-orchestrator, **disabled by default**, ports bound to 127.0.0.1 when enabled. |

### Wave 3: one deployment path, proved end to end

| ID | Repo | Task |
|---|---|---|
| **T13** | hermes-infra | Replace the bespoke replica bootstrap with `ansible-pull` of WOBI `site.yml` (`hermes_role: standby`). Switch replica IPs to DHCP reservations and do the cutover IP swap by reservation, not by editing netplan. Shrink the daemon's static range out of the DHCP pool. |
| **T14** | all | Build a replica from zero through the daemon, compare it with the live VM using the table in section 3 (services, ports, containers, config files), fix gaps, repeat until equal. Then Filip approves a cutover. |
| **T15** | crewai-orchestrator, VM | Archive the POC folders to a branch, prune merged `wt/t_*` branches, remove the dead `http-echo` containers and unused volumes on the VM (with Filip's OK for the VM part). |
| **T16** | D:\hermes | Update HERMES_MAP, READMEs and the `hermes-operations` skill so Hermes knows the VM is now code. |

Order: T1-T5 in parallel → T6-T12 in parallel → T13 → T14 → cutover (Filip approves) → T15-T16.

## 6. Defaults picked (change them if you disagree)

- WOBI is the canonical provisioning repo; terraforge-pipeline becomes dormant.
- The CrewAI stack stays as an optional role, off by default (Filip stopped it on 2026-10-06).
- Ansible runs on the VM (`ansible-pull`), not on Windows.
- Hermes data keeps coming from the nightly backup; Ansible owns everything else.
