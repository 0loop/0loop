# Contributing to ITS

ITS turns GitHub Actions runs into AI agent executions. This guide covers how changes get made here.

## Getting started

- Runtime: [Bun](https://bun.sh) 1.3.14 (pinned in CI).
- Install dependencies: `bun install --frozen-lockfile`.
- Checks: `bun run fmt:check`, `bun run lint:check`, `bun run build` (`dist/` is gitignored).

## Issue first

Every change starts with an issue, not code:

1. Open an issue using a template: **Issue** for bugs and questions, **Feature** for new functionality. State the type, scope, and impact; feature proposals include background, goals, and approach. Templates apply their labels automatically — contributors don't need (and can't) apply labels themselves.
2. Wait for a decision. The maintainer (or the repo's AI agent acting on their explicit authorization) applies a decision label:
   - `accepted` → development may start
   - `rejected` → closed, no development
   - `deferred` → not scheduled
     Decision labels belong to the maintainer side only — external contributors and automated processes never touch them.
3. Implement, then open a pull request that references the issue with a closing keyword (`fixes #N` / `closes #N`). The issue-gate check blocks pull requests that don't reference an accepted issue — there are no exemptions, docs included.

## Making changes

- The maintainer develops on `dev` and opens pull requests to `main`.
- External contributors: fork the repo, branch off the fork's `main` with a name like `<type>/<description>` (e.g. `fix/readme-typo`), and open a pull request from the fork branch to `main`.
- `main` receives no direct pushes; pull requests merge with squash, keeping history linear.
- Commit messages follow Conventional Commits: `<type>(scope): message`, imperative mood, describing only that change.
- A pull request body states what changed and why and references the accepted issue with a closing keyword (`fixes #N`); the issue-gate check enforces this.

## Developer Certificate of Origin (DCO)

ITS uses the DCO instead of a CLA. Every commit must carry a sign-off attesting you have the right to submit the change under the project's license:

```
Signed-off-by: Name <email>
```

- Sign when committing: `git commit -s` (or `--signoff`).
- Fix the last commit: `git commit --amend --no-edit --signoff`; several: `git rebase --signoff HEAD~N`.
- The GitHub web editor adds the trailer automatically.
- The sign-off identity must match the commit author; a CI check blocks pull requests with unsigned commits.

The full text is at https://developercertificate.org.

## Code style

- Comments explain why, not what. Prefer self-explanatory code over commented code.
- Modules are assembled from files: each module lives in its own directory and re-exports its public API from `index.ts`; import only from `index.ts`, never from internal files.
- Internal imports use the `@` alias (mapped to `src/`); no relative paths.
