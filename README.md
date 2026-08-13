# Newborn Mountain

A standalone GitHub Stacks / `gh stack` skill for creating, reviewing, rebasing,
merging, and retrospectively decomposing stacked pull requests.

`newborn-mountain` is the **stack skill**. It is separate from `sloppery-slope`,
the writing/orchestration skill.

## Layout

```text
newborn-mountain/
├── SKILL.md                         # frontmatter + core stack workflow
└── references/
    ├── commands.md                  # native command behavior details
    ├── stack-design.md              # native layer planning guidance
    └── troubleshooting.md           # native recovery guidance
```

## Frontmatter

```yaml
---
name: newborn-mountain
description: >-
  Use when creating, reviewing, rebasing, merging, checking out, or retrospectively
  decomposing stacked pull requests with GitHub Stacks / gh stack — "stack PRs",
  "create a PR stack", "split this branch into a stack", "gh stack", "stacked pull
  requests", "chain dependent PRs", "review a stack", "rebase a stack", or when a
  stack is already checked out.
---
```

## What it covers

- When a change is stack-shaped vs. a single PR.
- Greenfield stack creation with agent-safe `gh stack init/add/submit/view` forms.
- Retrospective decomposition of a finished branch into merge-safe layers.
- GitHub-native command behavior: TTY traps, JSON state, exit codes, remote config, sync/rebase/merge/link semantics.
- Shared-file ownership (`package.json`, lockfiles, locale files, generated registries, changesets).
- Review feedback on the owning branch, then cascade with `gh stack rebase --upstack --no-trunk` + `gh stack push`.
- Merge strategy: approve all layers and merge the top PR/stack number once for atomic landing, or merge safe lower segments early.
- GitHub UI merge box, merge queue behavior, CI stack metadata, and troubleshooting.

## Install

```sh
npx skills add grosspoetrysystems/newborn-mountain
```

Or copy this repo's `SKILL.md` and `references/` into a `newborn-mountain/` folder in your agent's skills directory (e.g. `.claude/skills/newborn-mountain/`).

## Source material

Reconciled from the current `stacked-pr-decomposition` working skill, GitHub's
native `github/gh-stack` skill, and GitHub's official stacked pull request docs:

- https://github.com/github/gh-stack/tree/main/skills/gh-stack

- https://docs.github.com/en/pull-requests/how-tos/stacked-pull-requests
- https://docs.github.com/en/pull-requests/reference/stacked-prs-cli-commands
- https://docs.github.com/en/pull-requests/how-tos/create-pull-requests/managing-stacked-pull-requests
- https://docs.github.com/en/pull-requests/how-tos/review-pull-requests/reviewing-stacked-pull-requests
- https://docs.github.com/en/pull-requests/how-tos/merge-and-close-pull-requests/merging-stacked-pull-requests
- https://docs.github.com/en/pull-requests/how-tos/merge-and-close-pull-requests/troubleshooting-stacked-pull-requests
