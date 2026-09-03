# Contributing to 0loop

0loop is intended to run AI agents in GitHub Actions workflows. This guide covers how changes get made here.

## Getting started

- Runtime: [Bun](https://bun.sh) 1.3.14 (pinned in CI).
- Install dependencies: `bun install --frozen-lockfile`.
- Checks: `bun run fmt:check`, `bun run lint:check`, `bun run build` (`dist/` is gitignored).

## Issue and design first

Every change starts with an issue that records the proposed design, not code:

1. Open an issue using a template: **Issue** for bugs and questions, **Feature** for new functionality. State the type, scope, and impact; feature proposals include background, goals, and approach. Issue titles carry a conventional type prefix like commit messages (`feat: `, `fix: `, `docs: `, `chore: `); the Feature template pre-fills `feat: ` and applies the `enhancement` label.
2. Wait for a decision. A repository maintainer applies a decision label:
   - `accepted` → implementation may start
   - `rejected` → closed, no development
   - `deferred` → not scheduled
     Decision labels belong to repository maintainers; contributors do not apply them.
3. After the design is accepted, implement it. Then open a pull request that references the issue with a closing keyword (`fixes #N` / `closes #N`). The issue-gate check blocks pull requests that don't reference an accepted issue — there are no exemptions, docs included.

## Making changes

- Fork the repository and use your fork as `origin` (the push destination). Add a second remote, with any name such as `upstream` or `0loop`, pointing to `0loop/0loop` (the canonical fetch source).
- Create a named branch from the canonical `main` using `<type>/<description>` (e.g. `fix/readme-typo`).
- Before submitting a pull request, fetch the canonical remote, rebase your branch, then push it to `origin`:

  ```bash
  git fetch <canonical-remote>
  git rebase <canonical-remote>/main
  git push -u origin <branch>
  ```

- Open the pull request from your fork branch to `0loop/0loop`'s `main`. Do not push directly to `main` or create merge commits; pull requests are integrated as one squashed commit to keep history linear.
- Commit messages follow Conventional Commits: `<type>(scope): message`, imperative mood, describing only that change.
- A pull request body states what changed and why and references the accepted issue with a closing keyword (`fixes #N`); the issue-gate check enforces this.

## Code style

- Comments explain why, not what. Prefer self-explanatory code over commented code.
- Modules are assembled from files: each module lives in its own directory and re-exports its public API from `index.ts`; import only from `index.ts`, never from internal files.
- Internal imports use the `@` alias (mapped to `src/`); no relative paths.
