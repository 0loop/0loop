---
name: github-workflow
description: >
  Git/GitHub collaboration rules for the ITS repo (minorcell/its). Follow this
  skill before any commit, branch, sync, rebase, PR, proposal (issue), or label
  operation. Core rules: never touch the main branch directly; the creator works
  on the dev branch, external contributors go through fork + PR; commits use the
  Conventional Commits format; sync with fetch + rebase before switching branches
  and before pushing to keep history linear; squash merges only; code changes
  require a proposal first (an issue labeled proposal:accepted), and PR bodies
  reference the issue with fixes/closes.
---

# GitHub Workflow

Git/GitHub collaboration rules for the ITS repo (minorcell/its). Check this document before any git operation, PR, proposal (issue), or label management.

## Hard rules

- Never touch the `main` branch directly: no direct commits, no direct pushes; main only receives changes through PRs.
- Always squash-merge (the repo is already configured to allow squash only), keeping history linear.
- External contributors must never push any branch to the main repo; the creator pushes `dev` only.

## Roles and paths

### Creator (repo owner)

Cannot fork their own repo, so development happens on the `dev` branch of the main repo:

1. Implement the feature on `dev`.
2. Commit and push `dev`.
3. Open a PR: `dev` → `main`, squash merge.

### External contributors

Must go through fork + PR:

1. Fork the main repo and clone the fork.
2. Create a clearly named branch: `<type>/<hyphenated-description>`, e.g. `feature/agent-log`, `fix/readme-typo`.
3. Commit and push to the fork.
4. Open a PR: fork branch → main repo `main`, squash merge.

## Commit message

Use the unified format `<type>(scope): message`:

- Common `type` values: `feat`, `fix`, `docs`, `chore`, `refactor`, `test`, `build`, `ci`
- `scope` is optional, e.g. `feat(agent): add triage tool`
- `message` in imperative mood; clear, human-readable, describing only this change, with no extraneous information

## Syncing (rebase, keep a straight line)

Sync main at two moments: **before switching to a new branch** and **before pushing**.

Creator (on `dev`):

```bash
git fetch origin
git rebase origin/main
```

External contributors (on a fork branch, `upstream` pointing at the main repo):

```bash
git fetch upstream
git rebase upstream/main
```

- Use fetch + rebase only; no merge commits, history always a straight line.
- On conflicts, resolve and `git rebase --continue`; never merge.

## Proposal first

Every code change: discuss the design first, then write code, then submit:

1. Submit a proposal as a GitHub issue, clearly stating background, goals, and approach.
2. Apply a type label to the issue: `proposal:full-spec` when the proposal changes a broad scope of the codebase, `proposal:minispec` when it is a small implementation or modification.
3. Wait for a decision label:
   - `proposal:accepted` → development starts
   - `proposal:rejected` → closed, no development
   - `proposal:deferred` → not scheduled
   - A proposal without a decision label must not start development
4. Open a PR when development is done. The PR body must reference the corresponding issue with a closing keyword (`fixes #N` or `closes #N`), so merging closes the issue automatically.

## PR body

No fixed template, but must state clearly:

- What changed and why;
- Background when necessary (context, tradeoffs);
- Reference the proposal issue (`fixes` / `closes`).

## Proposal labels

| Label                | Meaning                 |
| -------------------- | ----------------------- |
| `proposal:accepted`  | Accepted, dev starts    |
| `proposal:rejected`  | Rejected                |
| `proposal:deferred`  | Deferred, not scheduled |
| `proposal:full-spec` | Full specification      |
| `proposal:minispec`  | Mini specification      |

## Pre-push checklist

- [ ] Rebased on the latest `main`, history is a straight line
- [ ] Commit messages follow `<type>(scope): message`
- [ ] There is a `proposal:accepted` issue and the PR body references it with `fixes`/`closes` or `resolves` ...
- [ ] The PR body clearly states what changed and why
- [ ] The PR merges with squash
