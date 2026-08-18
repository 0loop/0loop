## Code Style Guidelines

Write comments sparingly, only when necessary. When a comment is needed, it explains **why**, not **what**. If code needs a comment to be understood, rewrite it until it is self-explanatory.

## Module Organization

Assemble modules from files. Each module lives in its own directory; each file exports its own functionality; the module's `index.ts` re-exports the module's full public API. Import only from a module's `index.ts`, never from its internal files.

## Imports

All internal imports use the `@` alias (mapped to `src/`); no relative paths.
