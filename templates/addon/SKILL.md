---
name: <name>
description: "<One sentence: when Hermes should load this skill.>"
version: 0.1.0
author: <who>, <date>
license: MIT
platforms: [linux]
metadata:
  hermes:
    tags: [<tag>, <tag>]
    related_skills: [hermes-operations]
---

# <Name>

## When to use

- <situations, in Filip's words where possible (Polish phrases he uses)>

## When not to use

- <cases that belong to another skill or need Filip's OK>

## How

```bash
<cmd> <subcommand> --flag value      # what it does
```

- Exit codes: 0 ok, 1 error, <n> <meaning>.
- Output to Filip: short, in Polish.

## Rules

- Never print secrets. Sending mail, deleting data or spending money needs Filip's OK.
