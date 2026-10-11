# Cutover to a rebuilt VM, then Hermes upgrading itself (plan, 2026-10-09)

Filip's ask (2026-10-08 22:11): keep the work of the other threads and of Hermes, pause Hermes before the
cutover and resume it on the new VM, sort out Tailscale/IP and rotations, apply WOBI to the live VM, then
cut over to a copy that does not differ from the original, and after that let Hermes upgrade itself
(and later every app Filip uses) within policies.

## 1. What the other threads changed on the live VM, and where it is kept now

| Thread | Change on unity-orchestrator | Kept by |
|---|---|---|
| Time management (timeplane, 2026-10-08) | `~/.hermes/scripts/timeplane`, `~/.hermes/autonomy/timeplane`, plugin `~/.hermes/plugins/timeplane`, skill, 4 Hermes cron jobs, `config.yaml` plugin entry, Google token with Tasks scope | backup (`~/.hermes`) |
| | `timeplane-intake.service`, `~/.local/bin/tp`, HA `secrets.yaml` intake token | **new WOBI role `timeplane`** (branch `iac/t17-live-sync`) |
| | HA `configuration.yaml` `packages:` include | **WOBI home_assistant role** (the old template dropped it) |
| | HA `packages/hermes_signals.yaml`, `hermes_signal.py` | backup format 2 (HA config tar) |
| | Code + 47 tests in `D:\hermes\_timeplane_work` (git, laptop only) | **not on GitHub** (pushing it was blocked: needs Filip's OK) |
| Syncthing hardening (2026-10-07) | discovery/relay/NAT off, laptop address pinned | WOBI syncthing role (already) |
| Phone / tmux (2026-10-06) | mosh, tmux, `tailscale set --ssh` | WOBI base + tailscale roles (already) |
| | tmux session `hermes` | not persistent (dies on reboot today too) |

Gaps found that are not from those threads but would lose work on a cutover:

* **Hermes' code work outside `~/.hermes`.** `~/crewai-orchestrator` has 29 commits on 20 local branches that are
  not on GitHub, 106 worktrees and 3 dirty files; `~/WOBI` 1 unpushed commit; `~/hermes-enhancements` and
  `~/honcho` have uncommitted changes; `~/hermes-kanban-repos`, `~/k8s_research`, `~/hermes-sandbox`,
  `~/decision_logs` are not git at all. None of this is in the backup.
* **Live backup is still format 1**: no HA config, no Syncthing identity, no bridge data. WOBI deploys format 2
  only when `hermes_backup_force_scripts: [hermes_backup.sh]`.
* **Identity that clients pin**: Tailscale node (100.66.185.39, `hermes-vm-1`; the phone's HA app and Termux
  aliases use the IP), Syncthing device ID (laptop pins it), SSH host keys (laptop and phone `known_hosts`).

## 2. Cutover design (proposal)

Principle stays: the new VM is **rebuilt** from base image + GitHub + Bitwarden + the encrypted backup. The
only change is that the last backup is taken **after Hermes is paused**, so nothing is lost in between, and
that it also carries the identities clients pin.

1. **Before (days ahead, reversible):**
   * Apply WOBI to the live VM (`--role active`) with `hermes_backup.sh` forced to format 2.
   * Backup format 3: add `extra/identity.tar` (`/var/lib/tailscale/tailscaled.state`, `/etc/ssh/ssh_host_*`)
     and `extra/work.tar` (for each git repo in `~`: `git bundle` of every ref not on the remote + a patch of
     uncommitted changes and untracked files per worktree; non-git work dirs as plain tar). Restore puts the
     identities back only when the role is promoted to active (standby never starts tailscaled).
   * WOBI role `workspaces`: clones Filip's repos into `~` (list in host_vars), then the restore applies the
     bundles and patches and recreates the worktrees of kanban tasks that are not done.
2. **Replica build**: daemon builds the copy as today, then runs the compatibility tests (section 4).
3. **Approval**: Filip approves in chat or the daemon terminal.
4. **Freeze on the old VM**: `hermes pause` (stops cron/kanban dispatch and new gateway turns), wait until no
   kanban task is `running` (or 30 min, then the in-flight worktree goes into `work.tar`), stop
   timeplane-intake, Syncthing after it is idle, HA.
5. **Final backup** on the old VM (`hermes_backup.sh --final`): format 3, uploaded, checksum recorded.
6. Old VM shuts down (kept 24 h). The daemon moves the DHCP reservation for **192.168.56.107** to the new
   VM's MAC and renews its lease (already built).
7. **New VM**: restore the final backup (state + work + identities), `pull.sh --role active`. tailscaled starts
   with the old node key, so the IP stays **100.66.185.39** and the name `hermes-vm-1`; Syncthing keeps its
   device ID; SSH host keys match, so laptop and phone see no warning.
8. `hermes resume` on the new VM; post-cutover tests; then finalize after 24 h or roll back (old VM back,
   reservation back, `hermes resume` there; anything done on the new VM in between is reported).

Fallback if the node key is not carried over: the new node joins with `TAILSCALE_AUTH_KEY`, gets a new
100.x IP, and the old node is removed in the Tailscale admin console and its IP reassigned to the new one.

## 3. Rotations and config changes

| Item | Why | Who |
|---|---|---|
| Bitwarden machine token | old one is in git history (hermes-config-backup skill, hermes-infra) | Filip creates a new one; I swap `~/.bws_token` and the daemon copy |
| GitHub PATs in VM git remote URLs | plain text in `.git/config` | WOBI base role switches to the Bitwarden credential helper; Filip revokes the old PATs |
| HA admin token Hermes issued itself (`ha_token.py`) | made without Filip's consent | Filip creates a long-lived token (`HASS_TOKEN` in Bitwarden), I disable the script |
| `HERMES_BRIDGE_AUTH_KEY`, `HERMES_API_SERVER_KEY` | missing in Bitwarden, the copy cannot start the bridge | Filip adds them (values from the live `.env`, or I generate new) |
| Syncthing GUI password file on the laptop | plain file | Filip moves it to Bitwarden |
| Tailscale | nothing to rotate if the node key is carried; otherwise a reusable `TAILSCALE_AUTH_KEY` must be valid | check before cutover |
| Phone HA app / Termux | use the IP 100.66.185.39; unchanged with the plan above | none |

## 4. Hermes upgrading itself (after the cutover works once)

1. **Detect**: a Hermes cron job (no LLM) checks NousResearch/hermes-agent releases and ScuraUrsa/hermes-agent,
   plus the pinned versions in WOBI (HA image, whisper/piper, Honcho, Ollama, Syncthing, Tailscale, Claude Code,
   bws), and writes `~/.hermes/autonomy/upgrades.json`.
2. **Prepare**: Hermes opens a WOBI branch `upgrade/<app>-<version>` that bumps the pin (host_vars), and asks
   the daemon for a replica built from that branch (`POST /v1/upgrades` with `wobi_ref`).
3. **Test**: the compatibility suite runs on the replica. It grows with everything built so far: gateway and
   CLI, `hermes doctor`, profiles, skills load, plugins (honcho, timeplane), cron jobs parse, `tp` offline tests
   and a dry tick, kanban, Honcho API, HA config check + intake token, bridge health, Syncthing folder, backup
   dry run. Failures go to a kanban card for the coder profile (Claude Code on the VM), which fixes the branch;
   the replica is rebuilt until green.
4. **Decide**: `policies.yaml` gets an `upgrades:` section per app: `auto` (patch versions, green tests,
   inside a maintenance window), `ask` (minor/major, anything touching secrets or networking), `never`.
   For `ask`, Hermes sends Filip the test report and waits.
5. **Cut over** with section 2, merge the WOBI branch, record in `decision_log.md`. Roll back automatically if
   post-cutover tests fail.
6. Later: the same loop for apps Filip uses outside the VM (laptop/phone apps) through their own update
   channels, with `ask` as the default.

## 5. Order of work

1. ✔ timeplane + HA packages into WOBI (`iac/t17-live-sync`).
2. Apply WOBI to the live VM (check mode, then real run; CrewAI skipped until Filip decides).
3. Backup format 3 (work + identities) and the `workspaces` role; test on a replica.
4. Compatibility test suite in the daemon (section 4.3), run on a fresh replica from tonight's backup.
5. Cutover rehearsal on two replicas (replica A as "old", replica B as "new"), then the real cutover with
   Filip's approval.
6. Upgrade detector + policies + first real upgrade (Hermes v0.20 → current) through the loop.
