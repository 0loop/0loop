# Contributing to ITS

ITS turns GitHub Actions runs into AI agent executions. This guide covers how changes get made here.

## Getting started

- Runtime: [Bun](https://bun.sh) 1.3.14 (pinned in CI).
- Install dependencies: `bun install --frozen-lockfile`.
- Checks: `bun run fmt:check`, `bun run lint:check`, `bun run build` (`dist/` is gitignored).

## Proposing changes

Every change starts with a proposal, not code:

1. Open an issue stating background, goals, and approach.
2. Add a type label: `proposal:minispec` for small changes, `proposal:full-spec` for broad ones.
3. Wait for a decision label. `proposal:accepted` means development may start; `proposal:rejected` and `proposal:deferred` mean it will not.

## Making changes

- The maintainer develops on `dev` and opens pull requests to `main`.
- External contributors: fork the repo, branch off the fork's `main` with a name like `<type>/<description>` (e.g. `fix/readme-typo`), and open a pull request from the fork branch to `main`.
- `main` receives no direct pushes; pull requests merge with squash, keeping history linear.
- Commit messages follow Conventional Commits: `<type>(scope): message`, imperative mood, describing only that change.
- A pull request body states what changed and why and references the proposal issue with a closing keyword (`fixes #N`).

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
