---
name: newborn-mountain
description: >-
  Use when creating, reviewing, rebasing, merging, or retrospectively decomposing
  stacked pull requests with GitHub Stacks / gh stack — "stack PRs", "create a PR
  stack", "split this branch into a stack", "gh stack", "stacked pull requests",
  "chain dependent PRs", "review a stack", "rebase a stack".
---

# Newborn Mountain

Newborn Mountain manages stacked pull requests: ordered, dependent PR layers where
the bottom PR targets trunk and each higher PR targets the branch below it. The
skill decides whether a change is stack-shaped, slices it into reviewable layers,
and uses GitHub Stack mechanics without losing merge safety.

**Core invariant:** if layer N depends on code, that dependency must live in layer
N or below it. Put shared types, schemas, utilities, flags, generated-file
ownership, and tests low enough that every upper layer compiles.

**Announce at start:** "I'm using Newborn Mountain to plan and operate this PR stack."

## When to use

Use when any are true:
- A feature spans multiple concerns: schema/data/API/UI/release, refactor + behavior, contract + consumers.
- A branch already contains a monolithic change and needs retrospective decomposition into reviewable stacked PRs.
- The user asks for `gh stack`, stacked pull requests, a chain of PRs, or a stack map.
- Review feedback lands mid-stack and must cascade upward.
- You need to merge/rebase/sync a stack safely.

Do not use when:
- The work is a single focused PR.
- The changes are unordered parallel PRs with no dependency chain.
- The work is cross-fork OSS: GitHub Stacks require all branches in one repository.
- The seams are unstable or exploratory; use a draft branch/spike before opening a stack.

## Decide the stack shape

A healthy stack has **2–5 ordered layers**, boring lower layers, and a one-sentence
dependency story per layer. If you cannot explain why PR N sits above PR N-1, the
split is wrong.

| Signal | Threshold | Action |
|---|---:|---|
| Review-relevant LOC | warn >250, flag >400 | split unless tests/generated churn dominate |
| Files per layer | warn >25 | check whether shared files/utilities belong lower |
| Stack depth | review carefully >5, exceptional >7 | collapse or restack |
| Adjacent file overlap | heavy overlap | assign ownership or move shared foundation down |
| Bottom rebase churn | repeated cascades | pause and restack |

Count review-relevant LOC, not raw churn. Tests, snapshots, generated files,
locale JSON, lockfiles, and changesets affect navigation but should not force a
bad seam by themselves.

## Stack styles

Pick the style whose bottom layer is most boring:
- **Refactor-first:** extract seam / rename / cleanup, then add behavior.
- **Migration-first:** expand contract/schema, dual path, cut over, cleanup.
- **API-contract-first:** interfaces/specs first, consumers above.
- **Feature-flagged:** flag plumbing, inactive path, implementation, enable, cleanup.
- **Branch-by-abstraction:** abstraction, route old callers, new impl, switch, remove old.

## Greenfield workflow

1. **Plan layers before branching.** For each layer: purpose, files, dependency
   sentence, ship-safety check, verification command, reviewer audience.
2. **Initialize the stack.**
   ```sh
   gh extension install github/gh-stack
   gh auth login
   gh stack init
   ```
   Use `--base <branch>` if trunk is not the repository default.
3. **Build bottom-up.** Commit the first layer, then add higher layers with:
   ```sh
   gh stack add <branch-name>
   ```
   Shortcut: `gh stack add -Am "message" <branch-name>` stages, commits, and adds a branch.
4. **Submit.**
   ```sh
   gh stack submit
   gh stack view
   ```
   `submit` pushes branches and creates/updates PRs linked as a GitHub Stack. New PRs default to ready in the interactive UI; with `--auto`, new PRs default to draft unless `--open` is passed.
5. **Gate every layer.** Each branch must compile without layers above it. Hide incomplete user-visible behavior behind a flag/shim.

## Retrospective decomposition

Use this when one branch already contains the work.

1. **Inventory the diff by concern, not commit chronology.** Group files into
   contracts/models, shared utilities, data plumbing, UI/behavior, tests, and cleanup.
2. **Place shared foundations low.** If two upper layers need the same utility,
   type, flag, or helper, move it to the lowest merge-safe layer.
3. **Create branches bottom-up.** Re-apply or cherry-pick only the files/commits
   for each layer. Split commits that span concerns.
4. **Keep every intermediate branch compiling.** A layer that only passes with
   unmerged higher code is not a layer.
5. **Register or link.**
   ```sh
   gh stack init layer-01-foundation layer-02-data layer-03-ui --base main
   gh stack submit
   ```
   If PRs already exist, use `gh stack link` with branches/PR numbers in bottom-to-top order.
6. **Write the stack map.** Every PR description should include its layer purpose,
   dependency sentence, shared-file ownership, ship-safety statement, and verification command.

## Shared file strategy

For `package.json`, lockfiles, locale files (`en.json`), route maps, generated
registries, and changesets, choose one owner:

| File type | Strategy |
|---|---|
| Dependencies / lockfiles | Layer that first imports the dependency owns both manifest and lockfile |
| Scripts/config | Layer that first uses the script/config owns it |
| Locale/route/generated registries | Either earliest consuming layer owns all related keys, or split by stable key namespace |
| Shared schemas/types/utilities | Lowest layer that can merge safely owns them |
| Cosmetic/generated churn | Defer to cleanup when it does not affect compile/runtime |
| Changesets | Use repo convention; for stacks that need every PR to carry one, use `none` for internal layers and one real bump where the user-visible aggregate lands |

Never let every layer casually touch the same shared file. That creates conflict
noise and hides the real seam.

**Verify shared-file diffs before pushing.** A rebase or partial re-apply of
`en.json`, lockfiles, or generated registries can silently drop unrelated entries.
Diff each shared file against the base (`git diff <base>...HEAD -- <file>`); it must
contain only your layer's keys. Restore accidental deletions byte-for-byte from
`main` — locale files are the common victim, where a stray deletion removes keys
other layers or pages still reference.

## Review feedback + rebase loop

When review feedback lands on a mid-stack PR, fix it on the branch that owns the
change — not on the top branch as a workaround. Cascade **upward only** so approved
lower layers are never disturbed.

```sh
gh stack checkout <branch>              # or gh stack up/down/top/bottom
# edit, test
git add .
git commit -m "helpful-message"
gh stack rebase --upstack --no-trunk    # regenerate SHAs for this layer and above only
gh stack push                           # per-branch --force-with-lease; lower layers are no-ops
```

**Preserve lower-layer approvals — do not re-parent onto trunk for every edit.**
Branch protection's "dismiss stale reviews on push" dismisses a layer's approval
whenever its head SHA changes. A plain `gh stack rebase` (or `gh stack sync`)
fetches trunk and re-parents the *whole* stack onto the latest `main`; if `main`
moved, that rewrites even approved layers **below** your change and dismisses their
approvals for no functional reason — a fix on layer 3 can drag layers 1 and 2 back
into review. `--upstack --no-trunk` touches only the changed layer and up. Reach for
a full trunk rebase only for a genuine `main` conflict or one deliberate pre-merge
catch-up, when you expect to re-collect approvals anyway.

**Lockfile / generated-file conflicts during a rebase:** never hand-merge
`pnpm-lock.yaml` or other generated files. Take the base side and regenerate:
`git checkout --ours pnpm-lock.yaml && pnpm install`, confirm
`pnpm install --frozen-lockfile` exits clean, `git add`, then `gh stack rebase --continue`.

If the repo requires signed commits, avoid the GitHub website **Rebase stack**
button; server-side rebases are unsigned. Use `gh stack rebase` locally, then
`gh stack push`.

## Merge strategy

Do **not** flatten a reviewed stack into one giant PR. The point is separate review
units with explicit dependencies.

**Review up, merge from the top.** Read and approve the stack bottom-up (layer 1 → N),
confirming each layer is green. Then, once the whole stack is approved and green,
merge **once from the top PR (N)** — not one layer at a time from the bottom.
Reviewing climbs up; the merge lands the stack from the top down in a single shot.
Any PR's merge box lands that PR plus every unmerged PR below it as one contiguous
bottom-up operation, so merging the **top** lands the entire stack atomically.

**Do not merge the bottom PR one layer at a time.** It feels right — layer 1 is
closest to `main` — but merging just the bottom PR retargets the layer above onto
`main` and rebases it, rewriting its head SHA. Branch protection's "dismiss stale
reviews on push" then dismisses that layer's approval, and the dismissal repeats for
every remaining layer as you crawl up: the same stale-review trap as a mid-stack
rebase (see **Review feedback + rebase loop** above). On a 5-stack that is four
needless re-reviews. Merging from the top avoids it — the whole approved group lands
together with no intermediate retarget.

**The GitHub web merge button is easy to misread.** Its label reflects only what that
PR's box will land, and nothing warns you it belongs to a stack:
- On the **bottom** PR (1/5) it reads **"Merge pull request"** (singular) — it merges
  only that one layer and silently starts the rebase cascade above.
- On the **top** PR (5/5) it reads **"Merge pull requests (5)"** — the count is the
  signal that one click lands all five. You see it only if you open the top PR.

**Prefer the CLI — it is the least ambiguous of all:**
```sh
gh stack merge      # lands the whole approved, green, linear stack in one operation
```
Make `gh stack merge` (or the **top** PR's merge box) the default; reserve the per-PR
web button for a deliberate partial merge (below).

Merge lower layers earlier only when they are independently useful and merge-safe:
foundation refactors, schema/contract expansion, inert flags, test scaffolding,
or cleanup that exposes no unfinished behavior. After any bottom/partial merge:

```sh
gh stack sync --prune
```

| Situation | Merge posture |
|---|---|
| User-visible feature must appear all at once | Approve all layers, then `gh stack merge` (or the top PR) once |
| Lower layer improves code safely by itself | Merge that bottom/mid contiguous segment early, then `gh stack sync --prune` |
| Lower layer exposes incomplete behavior | Keep unmerged or hide behind a flag/shim |
| Review uncovers a bad seam | Restack before merging; do not flatten as a shortcut |

## Troubleshooting

| Symptom | Fix |
|---|---|
| Merge box shows **Rebase stack** | Stack is not linear; run `gh stack rebase && gh stack push`, or website rebase if unsigned commits are acceptable |
| `gh stack rebase` conflicts | Resolve markers, `git add`, `gh stack rebase --continue`; use `--abort` to restore pre-rebase state |
| `gh stack sync` stops on conflict | It restores original branches; run `gh stack rebase`, resolve, then `gh stack push` |
| `gh stack modify` will not start | Need active stack, clean working tree, no rebase, no queued PR, linear history |
| Middle PR closed | Upper PRs are blocked; use `gh stack modify` or unstack/recreate |
| Merge queue ejects one PR | All PRs above are ejected too; fix cause and re-add stack |
| Large stack in merge queue | Queue may exceed group max by 50%; larger stacks split across groups |

For CI cost, use `github.event.pull_request.stack`: run expensive jobs on the
lowest unmerged PR (`stack.base.ref == pull_request.base.ref`) or top PR
(`stack.position == stack.size`).

## Author checklist

- Dependency story explains why every layer sits where it does.
- One concern per layer; no refactor + migration + behavior pileups.
- Bottom layer is boring and merge-safe.
- Incomplete behavior hidden behind flag/shim.
- Shared files have named owners.
- Every branch compiles without layers above it.
- Review feedback is committed on the owning branch, then rebased upward.
- After bottom merges, `gh stack sync --prune` runs before more work.

## Reviewer checklist

- Read the stack map first: foundation, mid-layer, or top?
- Judge the current layer against its stated purpose, not the whole future feature.
- Ask whether this layer could merge safely today.
- Request changes on the layer that owns the problem; do not patch lower-layer issues in a higher PR.
- Watch adjacent file overlap and repeated cascades; they signal a bad seam.
- Expect upper-layer CI to rerun after lower-layer fixes.

## Common mistakes

| Mistake | Fix |
|---|---|
| Hand-created PR bases but expected GitHub Stack UI | Use `gh stack submit` or `gh stack link` to register/link the stack |
| Treating retrospective split like greenfield planning | Work backwards from the existing diff; every reconstructed branch must compile |
| Fixing lower-layer review feedback on the top branch | Checkout owning branch, commit there, rebase/push upward |
| Website rebase in signed-commit repo | Rebase locally; server-side rebase commits are unsigned |
| Mid-stack PR merged alone | Impossible: contiguous segment from lowest unmerged PR always lands |
| Merging the bottom PR one layer at a time | Each bottom merge rebases the layer above → its approval is dismissed; merge from the top (`gh stack merge`) so the whole approved stack lands at once |
| Continuing after bottom merge without sync | Run `gh stack sync --prune` |
| Flattening after review | Keep layers as PRs; merge top once for atomic landing |
| Full `gh stack rebase`/`sync` per mid-stack fix, dismissing lower approvals | Moving trunk rewrites approved lower layers' SHAs → "dismiss stale reviews" fires; scope with `--upstack --no-trunk` |
| Hand-merging `pnpm-lock.yaml` conflict markers | Take base (`git checkout --ours`) + `pnpm install` to regenerate; verify `--frozen-lockfile`; then `--continue` |
| Rebase/re-apply silently drops unrelated `en.json`/generated keys | Diff each shared file against base before pushing; it must contain only your layer's keys; restore from `main` |

## Reference

Official docs:
- https://docs.github.com/en/pull-requests/how-tos/stacked-pull-requests
- https://docs.github.com/en/pull-requests/get-started/about-stacked-prs
- https://docs.github.com/en/pull-requests/get-started/stacked-prs-quickstart
- https://docs.github.com/en/pull-requests/reference/stacked-prs-cli-commands
- https://docs.github.com/en/pull-requests/how-tos/create-pull-requests/managing-stacked-pull-requests
- https://docs.github.com/en/pull-requests/how-tos/review-pull-requests/reviewing-stacked-pull-requests
- https://docs.github.com/en/pull-requests/how-tos/merge-and-close-pull-requests/merging-stacked-pull-requests
- https://docs.github.com/en/pull-requests/how-tos/merge-and-close-pull-requests/troubleshooting-stacked-pull-requests

Operational CLI notes: [references/gh-stack-commands.md](references/gh-stack-commands.md). It is a fast path, not an exhaustive manual; query the linked GitHub docs there for exact flags, exit codes, preview-limit changes, or full interactive UI behavior.
