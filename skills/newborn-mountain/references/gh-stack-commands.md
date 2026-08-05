# gh stack Command Reference

Full reference for `github/gh-stack` v0.1.0.

## Commands

### `gh stack init`

Initialize a new stack targeting the default branch.

```sh
gh stack init                           # from current branch
gh stack init branch1 branch2 branch3   # convert existing branches
```

Options:
- `--base <branch>` — target branch (default: repo's default branch)

### `gh stack submit`

Push all branches and create/recreate PRs for the stack.

```sh
gh stack submit
```

Each PR targets the previous branch in the stack. The bottom PR targets the base branch (e.g., `main`).

**This is the mandatory step.** `gh stack submit` registers the chain with GitHub's Stack metadata — Without it, GitHub sees standalone PRs, not a stack. No stack UI, no group-merge, no cascading rebase. Do not substitute `gh pr create` with manual base targeting.

### `gh stack rebase`

Pull from the remote and run a cascading rebase across the stack, from the trunk upward. This is the real command — there is **no `gh stack restack`**.

```sh
gh stack rebase                      # rebase whole stack onto latest trunk
gh stack rebase --upstack --no-trunk # rebase current branch and up, WITHOUT touching trunk
gh stack rebase --downstack          # only trunk..current
gh stack rebase --continue           # after resolving a conflict
gh stack rebase --abort              # restore all branches to pre-rebase state
```

On conflict it pauses and prints the conflicted files; resolve, `git add`, then `--continue`. `git rerere` (enabled by `gh stack init`) remembers resolutions across rebases. Prefer `--upstack --no-trunk` for mid-stack review fixes so you do not re-parent — and dismiss the approvals of — lower layers.

### `gh stack sync`

One-shot: fetch, reconcile local/remote stack, fast-forward trunk, cascade-rebase **if trunk moved**, push with `--force-with-lease`, and relink PRs.

```sh
gh stack sync
gh stack sync --prune   # also delete local branches for merged PRs
```

Use after a bottom/partial merge, or as a deliberate catch-up to `main`. Because it re-parents onto a moved trunk, expect it to dismiss stale approvals on lower layers — do not run it after every small edit.

### `gh stack view`

Show the branches in the stack with ordering and PR status. (There is **no `gh stack ls`**.)

```sh
gh stack view
gh stack view --short   # branch names only
gh stack view --json    # machine-readable
```

### `gh stack add`

Add a new branch to the top of an existing stack.

```sh
gh stack add <branch-name>
```

### `gh stack push`

Push active branches with a per-branch `--force-with-lease`. Branches whose SHA did not change are no-ops (so they do not trip "dismiss stale reviews").

```sh
gh stack push
```

### `gh stack merge`

Merge the stack bottom-up, up to and including a chosen PR. All-or-nothing; merge-queue aware.

```sh
gh stack merge          # interactive
gh stack merge 42       # up to PR 42
gh stack merge --yes --squash
```

### Removing / restructuring branches

There is **no `gh stack rm`**. To drop, reorder, fold, or insert branches, use `gh stack modify` (interactive; needs a clean, linear tree) or `gh stack unstack` to dissolve stack tracking while keeping the branches/PRs.

```sh
gh stack modify         # interactive restructure (drop with `x`)
gh stack unstack        # remove stack locally and on GitHub, keep branches
```

## Teardown warnings

### `gh pr close --delete-branch` is destructive

`--delete-branch` deletes both the remote and local branch ref. If you haven't backed up, the branch is gone.

Safe teardown order:
1. Back up branches: `git tag backup/<name> <branch>`
2. Verify backups: `git tag -l 'backup/*'`
3. Create and submit the correct `gh stack` replacement.
4. Verify the new stack's PRs are live.
5. Only then close old PRs: `gh pr close <PR-NUMBER> --delete-branch`

**Never delete before you've verified the replacement stack is created and submitted.**

## Changeset format gotchas

### `none` bump type — correct for intermediate layers

```yaml
---
"@aragon/app": none
---

Internal layer — no user-facing changes.
```

Parses correctly, passes CI, skips versioning, no CHANGELOG entry.

### Truly empty frontmatter — BROKEN

```yaml
---
---
```

Fails `@changesets/parse` v0.4.3+ with "expected a document, but the input is empty." Do not use.

## Tips

- **Commit messages** — follow your repo's conventions. `gh stack` doesn't override your commit discipline.
- **Force-push** — `gh stack submit` force-pushes branch updates. This is expected for stacked PRs (the branch history is part of the stack contract), but never force-push to `main`. If doing manual pushes between `gh stack` operations, use `--force-with-lease` (not bare `--force`) to avoid overwriting others' work on shared branches.
- **Multiple remotes** — if the repo has multiple remotes (common in worktrees), `gh stack submit` fails with "multiple remotes configured." Fix with `git config remote.pushDefault origin`.
- **Conflicts** — if two layers touch the same file, resolve on the layer that owns the change and cascade with `gh stack rebase` (`--upstack --no-trunk` for a mid-stack fix). For generated files (`pnpm-lock.yaml`), regenerate instead of hand-merging: `git checkout --ours`, `pnpm install`, verify `--frozen-lockfile`, `git add`, `gh stack rebase --continue`.
- **Merge order** — always merge bottom-up. Merging a middle PR out of order orphans the layers above it.

## Installation

```sh
gh extension install github/gh-stack
```

Verify:

```sh
gh extension list | grep stack
```