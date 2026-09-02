# 0loop

0loop is an early-stage GitHub Action for running AI agents in GitHub Actions
workflows. The intended design combines a natural-language task, model connection
settings, repository context, and GitHub permissions in one action step.

## Project status

0loop is currently a development scaffold, not a functional agent runner.

- `action.yml` declares the intended inputs and forwards them to the runtime.
- The composite action installs Bun 1.3.14, builds `src/index.ts`, and runs the
  resulting executable.
- The executable currently prints `0loop`. Model connections, event context, agent
  execution, GitHub tools, skills, and MCP integration are not implemented.

The examples below document the intended product interface. They do not work with
the current implementation and may change as each capability is implemented and
verified.

## Intended usage

### Issue triage

The intended issue-triage workflow classifies a new issue, applies a label, and
responds when more information is needed:

```yaml
name: Issue Triage
on:
  issues:
    types: [opened]

permissions:
  contents: read
  issues: write

jobs:
  triage:
    runs-on: ubuntu-latest
    steps:
      - uses: 0loop/0loop@v1
        with:
          prompt: |
            Classify the new issue as a bug, feature request, or question and
            apply the matching label. If a bug lacks reproduction steps, ask
            for them. Answer questions when the repository documentation
            provides a clear answer.
        env:
          ZEROLOOP_PROTOCOL: anthropic-messages
          ZEROLOOP_MODEL: your-model-id
          ZEROLOOP_API_KEY: ${{ secrets.ZEROLOOP_API_KEY }}
```

This design expects `ZEROLOOP_API_KEY` to be stored as a GitHub Actions secret. The
protocol and model can use repository or organization variables instead of
literals when they are managed centrally.

### Pull request review

A review workflow checks out the pull request so the agent can inspect its code:

```yaml
name: Pull Request Review
on:
  pull_request:
    types: [opened, synchronize]

permissions:
  contents: read
  pull-requests: write

jobs:
  review:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: 0loop/0loop@v1
        with:
          prompt: |
            Review the current pull request. Report actionable findings in the
            order correctness, security, and maintainability, then publish a
            concise summary.
        env:
          ZEROLOOP_PROTOCOL: openai-responses
          ZEROLOOP_MODEL: your-model-id
          ZEROLOOP_API_KEY: ${{ secrets.ZEROLOOP_API_KEY }}
```

### Scheduled digest

A scheduled workflow summarizes repository activity. GitHub Actions interprets
the cron expression as UTC unless a timezone is specified.

```yaml
name: Weekly Digest
on:
  schedule:
    - cron: '0 9 * * 1' # Monday at 09:00 UTC

permissions:
  contents: read
  discussions: write

jobs:
  digest:
    runs-on: ubuntu-latest
    steps:
      - uses: 0loop/0loop@v1
        with:
          prompt: |
            Summarize commits, pull requests, and issues from the previous seven
            days. Publish the digest to Discussions under progress, problems,
            and next steps.
        env:
          ZEROLOOP_PROTOCOL: gemini
          ZEROLOOP_MODEL: your-model-id
          ZEROLOOP_API_KEY: ${{ secrets.ZEROLOOP_API_KEY }}
```

### Compatible gateway

The intended interface separates the model protocol from the gateway URL. A
gateway must implement the selected protocol; claiming compatibility with a
provider does not guarantee that its protocol is identical.

```yaml
- uses: 0loop/0loop@v1
  with:
    prompt: |
      Read this repository's README and recent commits, then update incomplete
      documentation.
  env:
    ZEROLOOP_PROTOCOL: openai-chat-completions
    ZEROLOOP_MODEL: gateway-model-id
    ZEROLOOP_API_KEY: ${{ secrets.GATEWAY_API_KEY }}
    ZEROLOOP_BASE_URL: https://gateway.example.com/v1
```

### Local and remote skills

GitHub Actions passes action inputs as strings. List-valued inputs are intended
to use a YAML sequence inside a multiline string:

```yaml
- uses: 0loop/0loop@v1
  with:
    prompt: |
      Prepare a release according to the repository's release skill. Check the
      changelog, update versions, and open a release pull request.
    skills: |
      - .
      - github.com/owner/skills
  env:
    ZEROLOOP_PROTOCOL: anthropic-messages
    ZEROLOOP_MODEL: your-model-id
    ZEROLOOP_API_KEY: ${{ secrets.ZEROLOOP_API_KEY }}
```

The intended meaning of `.` is the current repository's `.agents/skills/`
directory. A `github.com/<owner>/<repo>` entry identifies a remote skill
repository to install through skills.sh.

### MCP server with credentials

MCP commands use the same planned list representation. Credentials remain in
the workflow environment and are referenced by name in the command:

```yaml
- uses: 0loop/0loop@v1
  with:
    prompt: |
      Query the internal statistics tool, reply to the issue with the relevant
      data, and apply the matching label.
    mcp: |
      - npx -y @team/stats-server --token $MCP_TOKEN
  env:
    ZEROLOOP_PROTOCOL: openai-responses
    ZEROLOOP_MODEL: your-model-id
    ZEROLOOP_API_KEY: ${{ secrets.ZEROLOOP_API_KEY }}
    MCP_TOKEN: ${{ secrets.MCP_TOKEN }}
```

## Intended interface

The target interface separates task behavior from model connection settings.
The composite action already declares these names, but the runtime does not yet
process their values.

Action inputs:

| Input    | Required | Intended meaning                                   |
| -------- | -------- | -------------------------------------------------- |
| `prompt` | Yes      | Natural-language description of the task           |
| `tools`  | No       | Additional tools made available to the agent       |
| `skills` | No       | Local or remote skills made available to the agent |
| `mcp`    | No       | MCP server commands started for the agent          |

Model connection environment:

| Variable            | Required | Intended meaning                                   |
| ------------------- | -------- | -------------------------------------------------- |
| `ZEROLOOP_PROTOCOL` | Yes      | Model API wire protocol                            |
| `ZEROLOOP_MODEL`    | Yes      | Model identifier accepted by the selected provider |
| `ZEROLOOP_API_KEY`  | Yes      | Credential for the selected provider               |
| `ZEROLOOP_BASE_URL` | No       | Base URL for a compatible gateway                  |

The planned protocol identifiers are `anthropic-messages`,
`openai-chat-completions`, `openai-responses`, and `gemini`. They name wire
protocols, not vendors. Provider and gateway compatibility will be documented
from implemented and tested behavior.

## Intended repository integration

0loop is intended to reuse common repository conventions instead of replacing
them with 0loop-specific equivalents:

- Load root-level `AGENTS.md` as repository instructions when it is present.
- Load local skills from `.agents/skills/` when `.` is listed in `skills`.
- Install explicitly listed remote skill repositories through skills.sh.
- Start explicitly configured MCP servers and make their tools available to the
  agent.

## Intended permission boundary

The workflow's `permissions:` block sets the requested access for its
`GITHUB_TOKEN`; GitHub may reduce the effective permissions. For example,
workflows triggered by a `pull_request` from a fork receive a read-only token and
do not receive repository secrets.

0loop is intended to expose a GitHub write operation only when the effective token
has the corresponding permission. This permission-aware tool selection is not
implemented yet.

## Development

The project uses Bun 1.3.14, which is pinned in CI.

```bash
bun install --frozen-lockfile
bun run fmt:check
bun run lint:check
bun run build
```

See [CONTRIBUTING.md](CONTRIBUTING.md) before proposing or implementing a
change.

## License

0loop is licensed under the [Apache License 2.0](LICENSE).
