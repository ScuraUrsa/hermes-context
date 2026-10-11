# AGENTS.md: Hermes context for agents from other projects

You are working on something related to **Hermes, Filip's personal 24/7 AI assistant**. This repo is the
single entry point: read this file, then the docs you need. You should not have to read the other ~30
ScuraUrsa repos to contribute.

## The 60-second version

- Hermes = Nous Research's open-source **Hermes Agent** (v0.20.0, upstream code pinned in WOBI) running in
  Ubuntu VMs on Filip's Windows laptop (VirtualBox).
  Goal: an assistant that replaces using the phone and the PC for everything that can be done virtually.
- Two VMs today: **prod** `192.168.56.107` (talks to Filip; Tailscale `100.66.185.39`) and **dev**
  `192.168.56.108` (repos, kanban board, workers, Claude Code). **Stage** is built on demand for upgrades.
- Everything inside a VM is code in **`ScuraUrsa/WOBI`** (`ansible/`, run on the VM with `pull.sh`;
  newest roles on branch `iac/final`). Hermes' data comes back from an encrypted nightly backup. Secrets are
  in Bitwarden Secrets Manager, read by key name at run time.
- Thinking runs on Ollama Cloud (DeepSeek V4 Pro / V4.1 Flash). **Coding runs only through the Claude
  Code CLI** on Filip's subscription (`cc_run.py`). No Anthropic API keys.
- Hermes works on its own: cron jobs + a kanban board + worker profiles (coder, reviewer, researcher,
  maintainer, writer, architect, …). Results are delivered as PRs with a review task.
- Filip speaks Polish. Answer him in Polish; write code, commits and repo docs in English.

## Read next

| If you need… | Read |
|---|---|
| What Hermes does for Filip, his rules | [docs/01-what-hermes-is.md](docs/01-what-hermes-is.md) |
| Components, VMs, models, data | [docs/02-architecture.md](docs/02-architecture.md) |
| Which repo holds what | [docs/03-repo-map.md](docs/03-repo-map.md) |
| How VMs are built, changed, cut over | [docs/04-deployment.md](docs/04-deployment.md) |
| Cron, kanban, autonomy loop, budget, health checks | [docs/05-operations.md](docs/05-operations.md) |
| Git/PR rules, decision logs, CI, safety rules | [docs/06-conventions.md](docs/06-conventions.md) |
| **Building a separate add-on product** | [docs/07-building-an-add-on.md](docs/07-building-an-add-on.md) + [templates/addon/](templates/addon/) |
| What is broken / pending right now | [docs/08-status-and-open-issues.md](docs/08-status-and-open-issues.md) |
| Where the facts came from, glossary | [docs/09-sources-and-glossary.md](docs/09-sources-and-glossary.md) |
| Original design documents | [docs/sources/](docs/sources/) |

## Hard rules (break none of them)

1. **No secrets anywhere** you write: Git, PRs, logs, Notion, chat. Use Bitwarden key names. Some old repos
   (`hermes-config-backup`, history of `hermes-infra`, `terraforge-modules`) hold leaked secrets: never
   read values out of them or copy them.
2. **Pull requests only.** Never push to a default branch. Filip merges to `main`/`master`.
3. **Prod is read-only for you** unless your task explicitly says otherwise. Build and test on dev, a
   replica (host daemon) or stage. Changes reach VMs through WOBI, never by hand edits alone.
4. **Nothing public**: bind to localhost, the host-only network or Tailscale.
5. **No OAuth grants, new tokens, outgoing mail, deletions of Filip's data, or spending** without Filip's
   explicit OK. Filip clicks OAuth consents himself.
6. **Coding through Claude Code CLI only**; never put a Claude token into Hermes' `anthropic` provider or
   name it `CLAUDE_CODE_OAUTH_TOKEN`.
7. Don't start the retired CrewAI stack. Don't create a second chat bot. Don't write Filip's Honcho memory
   from dev.
8. Leave a decision log for non-trivial changes, and register new features in WOBI
   `features/registry.yaml`.

## Keeping this repo true

This repo is a snapshot that must be kept current. When your work changes a fact stated here (a new repo,
a moved service, a new cron job, a new rule from Filip), update the relevant doc in the same PR series and
the Notion mirror ["Hermes — kontekst dla agentów"](https://app.notion.com/p/3f5ee453d19e8160b331ca802a9bc14e). Date every status claim.
