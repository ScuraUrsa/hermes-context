# 4. Deployment and lifecycle

## The chain

```
packer-images (optional)  ─► hermes-base OVA (Ubuntu 22.04 + WOBI base/docker), else the Ubuntu cloud image
terraforge-stacks/local   ─► VirtualBox VM: size, NICs, DHCP reservation, NoCloud seed (SSH key only)   [branch glados]
hermes-host-daemon        ─► the same VM creation for replicas / stage, then cutover with Filip's approval
WOBI ansible/pull.sh      ─► everything inside the VM (ansible-pull, run ON the VM, connection local)
nightly backup            ─► Hermes data (state), restored by role hermes_state
Bitwarden Secrets Manager ─► every secret, read at run time by key name
```

**WOBI is the single source of truth for what is inside a VM.** If something exists on a VM but not in
WOBI, it will be lost on the next rebuild. The Windows host has no Ansible; Ansible runs on the VM itself.

## WOBI Ansible

Repo `ScuraUrsa/WOBI`, directory `ansible/`. Note: the newest roles live on branch **`iac/final`**
(not yet merged into `master` on 2026-10-10); check which ref is deployed before changing anything
(`/var/lib/wobi/last-run.json` on the VM).

```bash
# run on the VM as root
sudo bash ansible/pull.sh --as <host> --role active|standby [--ref <branch|tag|sha>] [--tags a,b] [--check] [--diff] [-e k=v]
# examples
sudo bash ansible/pull.sh --as unity-orchestrator --role standby --tags base,docker --check   # dry run on a copy
sudo bash ansible/pull.sh --as unity-orchestrator --role active                              # the live prod
sudo bash ansible/pull.sh --as hermes-dev --role <dev role> --ref <branch> --tags coding_agents
```

- Hosts (`ansible/inventory/hosts.yml`): `unity-orchestrator` (prod identity), `hermes-dev` (dev).
  Host overrides in `inventory/host_vars/<host>/<role>.yml`; toggles `<role>_enabled` in
  `inventory/group_vars/all/<role>.yml` (dev turns off HA, bridge, Syncthing, timeplane, Honcho, Ollama).
- `hermes_role` `active` | `standby`: standby = gateway off, Hermes paused, no Tailscale join, no Syncthing
  pairing, no backup upload, nothing that reaches Filip's devices or accounts. `--role active` is refused
  on a machine whose hostname differs from `--as` unless `--force-active`.
- Roles (order in `site.yml`, `iac/final`): base, docker, bws, ollama, honcho, hermes, claude_code,
  coding_agents, hermes_state (tag `never`, restore from backup), hermes_backup (also ships all
  `~/.hermes/scripts`: budget.py, quota_governor.py, cc_run.py, gm_run.py, gh_actions_watchdog.py,
  hermes_work.py, …), home_assistant, notification_bridge, syncthing, tailscale, timeplane, kanban_bridge,
  phone_control, crewai (off), hermes_role (last; enforces active/standby and which cron jobs run where).
- Role rules: idempotent, check-mode safe, FQCN modules, `ansible-lint` production profile, no `meta`
  dependencies, variables prefixed `<role>_`, secrets only via `lookup('bws_secret', 'NAME')` with
  `no_log: true`.
- CI: `.github/workflows/ansible-ci.yml` (ansible-lint, syntax check, inventory check, shellcheck).
- Feature registry: `features/registry.yaml` says for each feature where it runs and how settled it is
  (`dev | stage | prod-experimental | prod-stable | retired`, review gate after 7 days). Hermes mirrors it
  to Notion "Funkcje Hermesa".

## Secrets

- Bitwarden Secrets Manager (EU cloud). Each VM has a machine-account token in `~/.bws_token` (0600),
  placed out of band, never by Git or cloud-init.
- Refer to secrets **by name only** (e.g. `GITHUB_TOKEN`, `HASS_TOKEN`, `HERMES_API_SERVER_KEY`,
  `HERMES_BRIDGE_AUTH_KEY`, `HERMES_KANBAN_API_TOKEN`, `CLAUDE_SETUP_TOKEN`, `OLLAMA_API_KEY`,
  `OLLAMA_API_KEY_FJ`, `TAILSCALE_AUTH_KEY`, `RCLONE_GDRIVE_CRYPT_PASSWORD`, `GOOGLE_<ACCOUNT>_TOKEN_JSON`).
- Never print `git remote -v` unredacted on the VMs (old clones carry PATs in remote URLs).
- Known leaks (history of `hermes-config-backup`, `hermes-infra`, `terraforge-modules`) are listed in
  [08-status-and-open-issues.md](08-status-and-open-issues.md). Do not read secrets out of them.

## Terraform, Packer, host daemon

- **terraforge-stacks**, branches `glados` / `iac/final`, dir `local/` (the daemon uses a host-local copy
  in `D:\hermes\terraforge-stacks\local` with generated keys, not in Git): Terraform + `manage-local-vm.ps1` for VirtualBox VMs
  (size, NICs, DHCP reservation). Default branch `aks_private` is the 2025 Azure work and is dormant.
- **terraforge-modules**: Azure modules, dormant (tracks an SSH private key, to be removed).
- **terraforge-pipeline**: Azure DevOps-era Ansible; legacy, harvested into WOBI.
- **packer-images**: optional `hermes-base` VirtualBox image (Ubuntu 22.04 + WOBI `base` and `docker`),
  only a speed-up; WOBI runs every role anyway.
- **hermes-infra / hermes-host-daemon**: Windows-side FastAPI service on `:9143` (`192.168.56.1:9143` from
  VMs). Builds replicas (`POST /v1/upgrades`), runs tests, cutover (`.../cutover` returns
  `403 approval_required` until Filip approves in chat or with `python approvals.py approve <token>`),
  moves the DHCP reservation of 192.168.56.107, keeps the old VM 24 h for rollback. Policy file
  `hermes-host-daemon/policies.yaml`. Doc: `hermes-infra/docs/self-upgrade-flow.md`.

## Flow of a change: dev → stage → prod

1. Code changes on dev (Hermes workers or Filip/Claude), as branches + PRs on GitHub.
2. A change meant for prod is pinned in WOBI (role defaults / host_vars point to the new commit or version).
3. Stage is built from zero from those refs, prod's latest backup is restored into it, compatibility tests
   run (gateway, `hermes doctor`, profiles, skills, plugins, cron parse, kanban, Honcho, HA, bridge,
   Syncthing, backup dry run).
4. Hermes asks Filip to try it; Filip approves in chat.
5. Cutover: pause Hermes, final backup (format 3, with identities and work), old VM off, restore on the new
   one, `hermes resume`, post-cutover tests, rollback window 24 h.

The stage pipeline is designed but not live yet (needs a Tailscale OAuth client). Today changes reach prod
by `pull.sh --role active` on prod after testing on dev or a replica, or by a daemon cutover.

## Access

- PC → VM: `ssh ubuntu@192.168.56.107` (prod) / `.108` (dev) with the key generated by terraforge-stacks
  (on the PC under `D:\hermes\terraforge-stacks\local\generated\`).
- Phone → prod: Tailscale `100.66.185.39` (`hermes-vm-1`), Tailscale SSH, mosh + tmux session `hermes`.
  Guide: Notion "Dostęp do Hermesa z telefonu (Termux + Tailscale + tmux)".
- Home Assistant: `http://100.66.185.39:8123` over Tailscale.
