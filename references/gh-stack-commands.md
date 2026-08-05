# gh stack operational reference

This is the agent's fast path for common `gh stack` work. It is **not** an
exhaustive manual. When you need exact flags, exit codes, current preview limits,
or full interactive UI behavior, query GitHub's official docs from the URLs below
instead of guessing.

Primary docs:
- https://docs.github.com/en/pull-requests/reference/stacked-prs-cli-commands
- https://docs.github.com/en/pull-requests/how-tos/stacked-pull-requests
> Stacked pull requests are in public preview and subject to change.

## Install

```sh
gh extension install github/gh-stack
gh auth login
```

Requirements: GitHub CLI, Git, and push access to the same repository. Cross-fork stacks are unsupported.

## Core workflow

```sh
gh stack init                 # initialize local stack tracking
gh stack add <branch>         # add a branch above the current top
gh stack push                 # push active stack branches
gh stack submit               # create/update PRs and GitHub Stack metadata
gh stack view                 # inspect order, PR links, status
```

Quick create:

```sh
gh stack init
gh stack add -Am "Add API contract" api-contract
gh stack add -Am "Add UI" ui-layer
gh stack submit
```

Use `--base <branch>` on `init` or `link` when trunk is not the repository default.

## Command summary

| Command | Purpose |
|---|---|
| `gh stack init [--base <branch>] [branches...]` | Start tracking a stack; can adopt/create named branches |
| `gh stack add [-A|-u] -m <msg> [branch]` | Add a branch on top; optional stage+commit shortcut |
| `gh stack view [--short|--json]` | Show stack order/status |
| `gh stack checkout [stack|pr|url|branch]` | Check out a local or remote stack/branch |
| `gh stack switch` | Interactive branch picker within current stack |
| `gh stack up/down/top/bottom/trunk` | Navigate relative to trunk/top/bottom |
| `gh stack submit [--auto] [--open]` | Push and create/update PRs + stack metadata |
| `gh stack push` | Push active stack branches; does not create/update PRs |
| `gh stack sync [--prune]` | Fetch, reconcile, rebase if needed, push, sync PR state, link open PRs, prune merged local branches |
| `gh stack rebase [--downstack|--upstack|--no-trunk]` | Cascading rebase across stack branches |
| `gh stack modify` | Interactive restructure: drop, fold, insert, reorder, rename |
| `gh stack link [--base <branch>] [--open] ...` | Link existing branches/PRs into a GitHub Stack without local tracking |
| `gh stack merge [stack|pr]` | Merge selected contiguous segment bottom-up |
| `gh stack unstack [--local]` | Remove stack tracking; preserves branches/PRs |
| `gh stack alias [name]` | Install short alias such as `gs` |

## Submit details

`gh stack submit` pushes all branches and creates/updates PRs. If no GitHub Stack exists, it links PRs into a stack. In interactive mode, it opens an editor where you can deselect branches, edit titles/descriptions, and choose draft/ready state.

With `--auto`, titles are generated and new PRs default to draft unless `--open` is passed.

If a stack is fully merged, it is complete and cannot be extended; submitting new branches creates a new stack rooted at trunk.

## Link existing branches or PRs

Use when branches/PRs already exist or another tool manages local branch state:

```sh
gh stack link feature-auth feature-api feature-ui
gh stack link 10 20 30
gh stack link https://github.com/owner/repo/pull/10 https://github.com/owner/repo/pull/20
gh stack link --base develop --open feat-a feat-b feat-c
```

Arguments are bottom-to-top. Existing PR bases are corrected to match the chain. Link is additive: it does not remove existing PRs from a stack.

## Rebase / sync

```sh
gh stack rebase                        # whole stack, re-parented onto trunk
gh stack rebase --upstack              # current branch and everything above
gh stack rebase --upstack --no-trunk   # current branch upward, do NOT fetch or re-parent trunk
gh stack rebase --downstack            # trunk up to current branch
gh stack rebase --no-trunk             # rebase stack branches onto each other, skip trunk fetch/rebase
gh stack rebase --continue             # after conflicts
gh stack rebase --abort                # restore pre-rebase state
gh stack push                          # per-branch --force-with-lease
```

**Preserve approvals after a mid-stack fix:** run `gh stack rebase --upstack --no-trunk`
from the changed branch. Rebasing the whole stack onto a moved trunk rewrites SHAs on
unchanged lower layers and force-pushes them, which trips "dismiss stale reviews on push"
and wipes valid approvals (a layer-3 fix can send layers 1–2 back to review). Re-parent
onto trunk only for a genuine `main` conflict or a deliberate pre-merge catch-up.

`gh stack sync` does more:
1. fetches remote changes,
2. reconciles remote/local stack shape,
3. fast-forwards trunk,
4. cascades rebase if trunk moved,
5. pushes rebased branches with `--force-with-lease`,
6. syncs PR state,
7. links open PRs into a stack if needed,
8. optionally prunes merged local branches.

Run after bottom merges:

```sh
gh stack sync --prune
```

If sync detects a real divergence, interactive mode offers: use remote as source of truth, delete the GitHub stack object, or cancel. Non-interactive mode aborts without pushing/updating.

## Modify / restructure

`gh stack modify` requires:
- active stack checked out,
- clean working tree,
- no rebase in progress,
- no PR queued to merge,
- linear commit history.

Operations:
- `x` drop branch from stack,
- `d` fold into branch below,
- `u` fold into branch above,
- `i` / `I` insert below/above,
- `r` rename,
- `Shift+↑` / `Shift+↓` reorder,
- `z` undo,
- save with Ctrl/Cmd+S.

After modifying an existing GitHub stack:

```sh
gh stack submit
```

## Merge

Merges are bottom-up and contiguous from the lowest unmerged PR.

```sh
gh stack merge              # choose interactively
gh stack merge 7            # merge stack #7 segment
gh stack merge 42           # merge up to PR #42
gh stack merge --yes --squash
```

The GitHub UI merge box can also merge the selected contiguous segment. Merging the top PR lands the whole stack. Merging a mid-stack PR lands it plus everything below; upper PRs remain open and retarget.

Auto-merge is unsupported for stacked PRs. Stacked merges cannot bypass requirements. Merge queues preserve stack order; large stacks may split across consecutive merge groups.

## Troubleshooting

| Problem | Recovery |
|---|---|
| Rebase conflict | Resolve markers, `git add`, `gh stack rebase --continue`; or `--abort` |
| Sync conflict | Sync restores original branches; run `gh stack rebase`, resolve, `gh stack push` |
| Modify will not start | Clean tree, finish/abort rebase, ensure no queued PR, rebase to linear history |
| Modify interrupted | `gh stack modify --abort`; or resolve, `git add`, `gh stack modify --continue` |
| PR cannot merge | Check all below approved/passing, current PR meets base rules, stack linear |
| Middle PR closed | Upper PRs blocked; restructure with `gh stack modify` or unstack/recreate |
| Signed commits required | Avoid website Rebase stack; use local `gh stack rebase` + `gh stack push` |
| Multiple remotes | Set push default: `git config remote.pushDefault origin` |

## CI metadata

Stack metadata is available at `github.event.pull_request.stack`:

| Expression | Meaning |
|---|---|
| `stack.number` | repository-scoped stack number |
| `stack.size` | number of PRs in stack |
| `stack.position` | 1-based position; bottom is 1 |
| `stack.base.ref` | ultimate stack base branch |
| `stack.base.sha` | base SHA |

Run expensive jobs only where needed:
- lowest unmerged PR: `github.event.pull_request.stack.base.ref == github.event.pull_request.base.ref`
- top PR: `github.event.pull_request.stack.position == github.event.pull_request.stack.size`

## Safety notes

- Prefer `gh stack push` / `sync`; when pushing manually after rebase, use `--force-with-lease`, never bare `--force`.
- Do not delete old PR branches until the replacement stack exists and is verified.
- `gh pr close --delete-branch` deletes the remote and local branch ref. Back up first if reconstructing a mistaken stack.
