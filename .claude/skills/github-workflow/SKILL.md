---
name: github-workflow
description: >
  Git/GitHub collaboration rules for the ITS repo (minorcell/its). Follow this
  skill before any commit, branch, sync, rebase, PR, issue, or label
  operation. Core rules: never touch the main branch directly; maintainers and
  collaborators with write access use named branches from origin/main, while
  external contributors use a fork; commits use the Conventional Commits format;
  sync with fetch + rebase before creating a branch and before pushing to keep
  history linear; squash merges only; every change is issue-driven — a PR must
  reference an issue carrying the accepted label, and decision labels
  (accepted/rejected/deferred) are applied only by the maintainer side: the
  maintainer, or the repo's AI agent on their explicit authorization.
---

# GitHub Workflow

Git/GitHub collaboration rules for the ITS repo (minorcell/its). Check this document before any git operation, PR, issue, or label management.

## Hard rules

- Never touch the `main` branch directly: no direct commits, no direct pushes; main only receives changes through PRs.
- Always squash-merge (the repo is already configured to allow squash only), keeping history linear.
- Only maintainers and collaborators with write access push feature branches to the main repository; external contributors push to their forks.

## Roles and paths

### Maintainers and collaborators with write access

Create a named branch directly from the latest `origin/main`:

1. Fetch the main repository: `git fetch origin`.
2. Create a branch: `git switch -c <type>/<hyphenated-description> origin/main`.
3. Commit and push the branch to `origin`.
4. Open a PR from the branch to `main`, then squash merge.

### External contributors without write access

Must go through fork + PR:

1. Fork the main repo and clone the fork.
2. Configure the main repository as `upstream` and fetch it.
3. Create a clearly named branch from `upstream/main`: `<type>/<hyphenated-description>`, e.g. `feature/agent-log`, `fix/readme-typo`.
4. Commit and push the branch to the fork.
5. Open a PR from the fork branch to the main repository's `main`, then squash merge.

## Commit message

Use the unified format `<type>(scope): message`:

- Common `type` values: `feat`, `fix`, `docs`, `chore`, `refactor`, `test`, `build`, `ci`
- `scope` is optional, e.g. `feat(agent): add triage tool`
- `message` in imperative mood; clear, human-readable, describing only this change, with no extraneous information

## Syncing (rebase, keep a straight line)

Fetch the canonical repository before creating a branch. Rebase a branch on the latest `main` before pushing.

Maintainers and collaborators with write access:

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

## Issue first

Every change: issue first, then code, then PR:

1. Open an issue using a template: **issue** for bugs and questions, **feature** for new functionality, or blank. State the type, scope, and impact; feature proposals include background, goals, and approach. Issue titles carry a conventional type prefix like commit messages (`feat: `, `fix: `, `docs: `, `chore: `); the feature template pre-fills `feat: `. Templates apply their labels automatically — contributors don't need (and can't) apply labels themselves.
2. Wait for the maintainer's decision:
   - `accepted` → development starts
   - `rejected` → closed, no development
   - `deferred` → not scheduled
     Decision labels are applied only by the maintainer side: the maintainer, or the repo's AI agent (Claude Code / ITS) when the maintainer explicitly authorizes it. External contributors and automated triage never touch them. No substitute counts as acceptance: not answers in conversation, not "go ahead" — only the label on the issue. Without it, development must not start.
3. Open a PR when development is done. The PR body must reference the issue with a closing keyword (`fixes #N` or `closes #N`), so merging closes the issue automatically. The issue-gate check enforces this — no exemptions, docs included.

## PR body

No fixed template, but must state clearly:

- What changed and why;
- Background when necessary (context, tradeoffs);
- Reference the accepted issue (`fixes` / `closes`).

## Labels

| Label      | Meaning                      |
| ---------- | ---------------------------- |
| `accepted` | Accepted, development starts |
| `rejected` | Rejected                     |
| `deferred` | Deferred, not scheduled      |

Decision labels (`accepted`, `rejected`, `deferred`) are applied only by the maintainer side: the maintainer, or the repo's AI agent on the maintainer's explicit authorization. Type labels (`bug`, `enhancement`, `documentation`, `question`) are triage metadata; the feature template applies `enhancement` automatically, and the maintainer adds other type labels during triage.

## Pre-push checklist

- [ ] Rebased on the latest `main`, history is a straight line
- [ ] Commit messages follow `<type>(scope): message`
- [ ] The PR body references an issue carrying `accepted` with a closing keyword (the issue-gate check enforces this)
- [ ] The PR body clearly states what changed and why
- [ ] The PR merges with squash
