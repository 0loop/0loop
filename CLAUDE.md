## Collaboration

All changes are issue-driven: open an issue via a template (issue / feature) and wait for the maintainer to apply the `accepted` label before starting development. Pull requests must reference an accepted issue with a closing keyword (`fixes #N`); the issue-gate CI check enforces this with no exemptions. Decision labels (`accepted` / `rejected` / `deferred`) belong to the maintainer side: the maintainer, or this AI agent when the maintainer explicitly authorizes it. See CONTRIBUTING.md and the github-workflow skill for the full workflow.

## Code Style Guidelines

Write comments sparingly, only when necessary. When a comment is needed, it explains **why**, not **what**. If code needs a comment to be understood, rewrite it until it is self-explanatory.

## Module Organization

Assemble modules from files. Each module lives in its own directory; each file exports its own functionality; the module's `index.ts` re-exports the module's full public API. Import only from a module's `index.ts`, never from its internal files.

## Imports

All internal imports use the `@` alias (mapped to `src/`); no relative paths.
