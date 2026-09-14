# Contributing to 0loop

0loop is intended to run AI agents in GitHub Actions workflows. This guide covers how changes get made here.

## Getting started

- Runtime: [Bun](https://bun.sh) 1.3.14 (pinned in CI).
- Install dependencies: `bun install --frozen-lockfile`.
- Checks: `bun run fmt:check`, `bun run lint:check`, `bun run build` (`dist/` is gitignored).

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
- A pull request body states what changed and why.

## Code style

- Comments explain why, not what. Prefer self-explanatory code over commented code.
- Modules are assembled from files: each module lives in its own directory and re-exports its public API from `index.ts`; import only from `index.ts`, never from internal files.
- Internal imports use the `@` alias (mapped to `src/`); no relative paths.
