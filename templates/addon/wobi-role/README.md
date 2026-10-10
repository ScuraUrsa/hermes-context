# WOBI role template for an add-on

Copy into `WOBI/ansible/roles/<name>/` and model it on `roles/timeplane` (branch `iac/final`).

```
defaults/main.yml   <name>_repo, <name>_version (pinned commit), <name>_files (repo path -> VM path), unit names
tasks/main.yml      git checkout at the pin -> copy missing files (keep differing live files unless
                    <name>_overwrite) -> units/launchers -> secrets via lookup('bws_secret', 'KEY') with
                    no_log -> start services only when hermes_role == 'active'
handlers/main.yml   restart handlers
meta/main.yml       galaxy_info, dependencies: [] (order comes from site.yml)
templates/          <name>.service.j2, launcher .j2
```

Also add:

- `ansible/inventory/group_vars/all/<name>.yml` → `<name>_enabled: true|false`
- `ansible/inventory/host_vars/<host>/<name>.yml` for per-VM overrides (dev disables Filip-facing roles in `toggles.yml`)
- the role to `ansible/site.yml` (shared file: mention it in the PR)
- cron job names to `hermes_role_cron_paused` / `hermes_role_cron_resumed` of each host

Checks: `ansible-lint` (production profile), run twice on dev/replica with no changes on the second run,
`--check` mode works.
