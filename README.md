# hermes-context

**One place that explains Hermes**, Filip's personal 24/7 AI assistant: what it is for him, how it is
built, deployed and operated, and how to contribute a separate add-on product without reading every
ScuraUrsa repository.

Jedno miejsce z pełnym kontekstem Hermesa dla agentów z innych projektów. Wersja dla ludzi (po polsku):
[Notion → Hermes Organized → „Hermes — kontekst dla agentów”](https://app.notion.com/p/3f5ee453d19e8160b331ca802a9bc14e).

**Start with [AGENTS.md](AGENTS.md).**

```
AGENTS.md                         the brief every agent reads first (CLAUDE.md imports it)
docs/01-what-hermes-is.md         Filip's context, use cases, standing rules
docs/02-architecture.md           VMs, services, Hermes concepts, models, data
docs/03-repo-map.md               every ScuraUrsa repo in one line
docs/04-deployment.md             WOBI/Ansible, Terraform, Packer, host daemon, dev → stage → prod
docs/05-operations.md             cron, kanban, autonomy loop, budget, Claude Code, Google, backups
docs/06-conventions.md            PRs, branches, decision logs, CI, safety
docs/07-building-an-add-on.md     how to build and ship a separate product for Hermes
docs/08-status-and-open-issues.md snapshot of what is broken and pending
docs/09-sources-and-glossary.md   where each fact comes from; terms
docs/sources/                     original design documents (2026-10-07..09)
templates/addon/                  AGENTS.md, SKILL.md, WOBI role and registry templates for add-ons
```

Snapshot date: **2026-10-11**. Status claims are dated; verify them against the linked sources.
