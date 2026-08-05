# Newborn Mountain

A [`gh stack`](https://github.com/github/gh-stack) skill — manages stacked PR workflows for layered feature delivery. Covers both greenfield stacks and retroactive decomposition of monolithic branches.

```
skills/
└── newborn-mountain/
    ├── SKILL.md                       # frontmatter + core workflow
    └── references/
        └── gh-stack-commands.md      # full command reference + teardown warnings
```

## SKILL.md

```yaml
---
name: newborn-mountain
description: Scaffold stacked PR workflows with gh stack — init, submit, rebase, sync, decompose.
---
```

Structured per the [Agent Skills](https://github.com/vercel-labs/skills) spec: `SKILL.md` with portable `name` + `description` frontmatter, supporting files in `references/` loaded on demand.

## What it covers

- Greenfield stack creation (plan → init → submit → sync)
- Retroactive decomposition of a finished monolithic branch into layers
- Mid-stack review fixes that preserve lower-layer approvals (`gh stack rebase --upstack --no-trunk`)
- Cross-layer shared file strategy (`package.json`, lockfiles, locale files, changesets)
- Changeset choreography for stacked PRs (none bump type vs real bump on final layer)
- Teardown guardrails for wrongly-created PRs
- Critical enforcement: use `gh stack`, never hand-create stacked PRs

## Install

```sh
npx skills add grosspoetrysystems/newborn-mountain
```

Or copy `skills/newborn-mountain/` into your repo's skills directory.