---
name: test
description: Write, update, review, and run focused automated tests for TypeScript and JavaScript projects using Bun's built-in test runner and mock/spy utilities. Use for behavior changes, bug regressions, API client tests, protocol adapters, streaming parsing, async state, test failures, mocks, coverage questions, and test reviews. Do not use for browser E2E, or for projects whose configured runner is Jest, Vitest, or another test framework.
---

# Test with Bun

Protect stable, observable product behavior. Keep tests focused enough to survive implementation-preserving refactors.

## Establish project context

1. Read the project's `AGENTS.md` before planning or editing.
2. Inspect the behavior owner, its public dependencies, and nearby code before choosing a test boundary.
3. Treat these files as the test infrastructure contract:
   - `package.json`: scripts, Bun version, dependencies that affect test behavior.
   - `bunfig.toml`: `[test]` settings such as preload, root, coverage, and test patterns, when present.
   - `tsconfig.json`: strictness, module resolution, and types that affect test compilation and execution.
4. Keep this skill scoped to the current project's Bun test setup. For other runners (Jest, Vitest, Node's built-in test runner), inspect their configuration instead of applying Bun commands or APIs.
5. Use a shared helper skill when a test exercises or mocks a shared contract from another package or library.

## What matters most in this project

ITS is an async agent runtime that runs headless on GitHub runners. Prioritize tests around:

- Protocol adapters: request serialization, response parsing, streaming, and error mapping for each wire protocol (anthropic-messages, openai-chat-completions, openai-responses, gemini).
- Event context parsing: GITHUB_EVENT_PATH payloads (issue, PR, schedule) and their validation.
- The GitHub API client: URL, method, headers, and body construction under GITHUB_TOKEN.
- Permission gating: how the toolset expands and contracts with the workflow's `permissions:` declaration.
- Idempotency: detecting an already-handled event so a re-triggered run does not act twice.

## Decide whether to add a test

Add or update a test when a change affects a reachable contract such as:

- An agent-visible behavior: a tool call, its result, and the resulting state or side effect (comment, PR, label).
- Request construction, response parsing, streaming, retries, credentials, or error mapping at any protocol or gateway boundary.
- A reusable business rule with meaningful inputs and outputs: permission gating, idempotency, prompt assembly, event parsing.
- Persistence, ordering, or data flow across an async run.
- A bug whose regression can be reproduced through a public boundary.

Do not add a test only because:

- A module, branch, or file exists.
- A coverage report marks a line uncovered.
- TypeScript already excludes the input.

Treat coverage as a diagnostic signal, not an acceptance percentage.

## Choose the smallest useful boundary

- Test pure transformations, validation, business rules, and parsers directly.
- Test API client modules through their exported functions. Mock `fetch` or the underlying transport, then assert the URL, method, headers, body, parsed result, or typed error that represents the public contract.
- Test protocol adapters through their exported functions: feed a recorded response or stream, assert the parsed output and the next request the adapter produces.
- Mock the gateway client when testing an agent step that consumes it. Keep the real client when testing request construction or response parsing.
- Do not export private helpers, split production code, or introduce dependency injection solely to make an implementation detail testable.
- Do not duplicate behavior already owned by the runtime, framework, or protocol SDK. Test this project's integration, overrides, and regressions.

The current Bun test setup cannot faithfully prove real GitHub API behavior (auth, permissions, rate limits), real model gateway behavior, or the GitHub runner environment itself. For those contracts, report the gap and rely on a real workflow run to validate them. Do not add another runner or test library without a demonstrated project-level need and explicit task scope.

## Place and name specs correctly

- Place a spec beside the module it owns and name it `<subject>.test.ts`.
- Keep one test file per coherent owner-level contract. Do not create one test file per source file by default.
- Follow the project's existing test file conventions if they differ. Bun's default discovery includes `.test.ts` and `.spec.ts`, but `bunfig.toml` may override this.
- Keep shared test setup, fixtures, and resource mocks in a dedicated test support directory, not next to production code, unless project convention says otherwise.

## Assert behavior instead of implementation

- Drive state through exported functions, callbacks, timers, or mocked boundary responses.
- Assert serialized requests, parsed responses, returned values, surfaced errors, and external side effects.
- Avoid asserting internal state, private method calls, or incidental object shape.
- Avoid snapshots unless the serialized output is intentionally public and stable.
- Keep one behavior per test. Use multiple assertions only when they jointly prove that behavior.
- Use valid fixtures by default. Override only fields relevant to the scenario.
- For protocol-sensitive fixtures, verify them against the protocol definition and backend implementation. Raise a mismatch instead of encoding divergent behavior in a test.

## Mock real boundaries

- Never make a real network request from a test.
- Mock `fetch` only when testing the API client itself.
- Mock the exported gateway or API client, filesystem, or other external service boundaries when testing their consumers.
- Keep production code that transforms the asserted behavior real.
- Keep a mock faithful to the public contract used by the test, including rejected promises and error shapes where relevant.
- Keep one-off mocks in the spec. Add a shared mock under test support only after multiple suites need the same faithful boundary.
- Use Bun's `mock`, `spyOn`, and `mock.module` utilities from `bun:test`. Reset or restore mocks in `beforeEach`/`afterEach` when assertions depend on call history or global behavior.

## Control async behavior and isolation

- Await promises and async completion. Assert observable completion instead of using fixed sleeps.
- Use fake timers only when timer behavior is part of the contract. Restore real timers after the test.
- Control time, randomness, fetch responses, and shared stores so the test is deterministic.
- Restore modified globals, mocks, and timers in the same suite that changed them.
- Reuse shared test setup already provided by the project. Add a missing stub locally unless several suites genuinely share it.

## Follow the implementation workflow

1. State the observable contract and realistic regression risk.
2. Select the smallest boundary containing the behavior owner.
3. Establish a failing regression case before changing production behavior when practical.
4. Implement one coherent scenario.
5. Run that spec and fix failures before expanding scope.
6. Run only the affected checks needed for confidence.
7. Remove redundant assertions, implementation-coupled traversal, and unnecessary mocks.

Do not add a test file until its owner, boundary, and expected failure mode are clear.

## Run focused commands

Run commands from the project root. Bun test commands:

```bash
# Run a focused test file
bun test src/path/to/subject.test.ts

# Run a focused test file in watch mode
bun test --watch src/path/to/subject.test.ts

# Run the full test suite with coverage; use only when justified
bun test --coverage

# Run all tests matching a pattern
bun test api-client

# TypeScript check for affected production/test code
bunx tsc --noEmit

# Lint check for affected production/test code (read-only)
bun run lint:check
```

Do not use build or deploy commands for test verification if they trigger side effects or require credentials. Report commands not run and the reason.

## Review the result

Confirm all of the following:

- Each test protects a reachable product contract or a meaningful regression.
- The behavior is exercised through a public boundary.
- The test owns the correct layer and mocks only external dependencies.
- Async work, globals, timers, and mock state are deterministic and cleaned up.
- The test would survive an implementation refactor that preserves behavior.
- The reviewer can name the realistic regression and the assertion that catches it.
- The reported test category matches reality; a Bun test runner spec is not a real GitHub API or real gateway integration test.
