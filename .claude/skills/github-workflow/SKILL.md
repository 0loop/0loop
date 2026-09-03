---
name: github-workflow
description: >
  Git/GitHub collaboration rules for the 0loop repo (0loop/0loop). Follow this
  skill before any commit, branch, sync, rebase, PR, issue, or label
  operation. Core rules: use the fork + issue + pull request workflow; `origin`
  must point to the contributor's fork and another remote (name unrestricted)
  must point to `0loop/0loop`; commits use the Conventional Commits format;
  sync with fetch + rebase before creating a branch and before pushing to keep
  history linear; every change is issue-driven — a PR must
  reference an issue carrying the accepted label; decision labels
  (accepted/rejected/deferred) are applied by repository maintainers.
---

# GitHub Workflow

Git/GitHub collaboration rules for the 0loop repo (0loop/0loop). Check this document before any git operation, PR, issue, or label management.

## Hard rules

- Never touch the `main` branch directly: no direct commits, no direct pushes; main only receives changes through PRs.
- Never use `git merge` or create merge commits. Keep branch history linear; the maintainer integrates each PR as one squashed commit.
- Use a fork for every change. Push feature branches only to `origin`; the `0loop/0loop` repository receives changes through pull requests.

## Fork setup and workflow

The checkout must have at least these remotes:

- `origin`: the contributor's fork (push destination).
- A second remote with any name (commonly `upstream`): the canonical `0loop/0loop` repository (fetch source).

Create a clearly named branch from the canonical repository's `main`: `<type>/<hyphenated-description>`, e.g. `feature/agent-log`, `fix/readme-typo`.

Every change follows this order: fork, issue/design, maintainer acceptance, implementation, then pull request.

## Commit message

Use the unified format `<type>(scope): message`:

- Common `type` values: `feat`, `fix`, `docs`, `chore`, `refactor`, `test`, `build`, `ci`
- `scope` is optional, e.g. `feat(agent): add triage tool`
- `message` in imperative mood; clear, human-readable, describing only this change, with no extraneous information

## Syncing (rebase, keep a straight line)

Before pushing or opening a PR, fetch the canonical remote, rebase on its latest `main`, and push the rebased branch to `origin`:

```bash
git fetch <canonical-remote>
git rebase <canonical-remote>/main
git push -u origin <branch>
```

- Use fetch + rebase only; no merge commits, history always a straight line.
- On conflicts, resolve and `git rebase --continue`; never merge.

## Issue and design first

Every change starts with an issue that records the proposed design; do not write implementation code until that design is accepted:

1. Open an issue using a template: **issue** for bugs and questions, **feature** for new functionality, or blank. State the type, scope, and impact; feature proposals include background, goals, and approach. Issue titles carry a conventional type prefix like commit messages (`feat: `, `fix: `, `docs: `, `chore: `); the feature template pre-fills `feat: `. Templates apply their labels automatically — contributors don't need (and can't) apply labels themselves.
2. Wait for the maintainer's decision:
   - `accepted` → implementation may start
   - `rejected` → closed, no development
   - `deferred` → not scheduled
     Decision labels are applied by the repository maintainers. No substitute counts as acceptance: not answers in conversation, not "go ahead" — only the label on the issue. Without it, development must not start.
3. After implementation is complete, open a PR from the `origin` branch to `0loop/0loop`'s `main`. The PR body must reference the accepted issue with a closing keyword (`fixes #N` or `closes #N`), so PR integration closes the issue automatically. The issue-gate check enforces this — no exemptions, docs included.

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

Decision labels (`accepted`, `rejected`, `deferred`) are applied by the repository maintainers. Type labels (`bug`, `enhancement`, `documentation`, `question`) are triage metadata; the feature template applies `enhancement` automatically, and maintainers add other type labels during triage.

## Pre-push checklist

- [ ] Rebased on the latest `main`, history is a straight line
- [ ] Commit messages follow `<type>(scope): message`
- [ ] The PR body references an issue carrying `accepted` with a closing keyword (the issue-gate check enforces this)
- [ ] The PR body clearly states what changed and why
- [ ] The maintainer integrates the PR as one squashed commit
