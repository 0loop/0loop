# ITS

**Turn GitHub Actions into your AI agent.**

ITS is a GitHub Action that turns every workflow run into an AI agent execution: GitHub provides the environment (a fresh Linux VM, checked-out code, an auto-injected GITHUB_TOKEN), you write a single `prompt`, and the agent reads code, runs commands, and edits files in that environment — then writes the results back to the repo using GitHub's own permission mechanisms: comments, PRs, labels.

- **Zero infrastructure**: free quota on public repos, no servers of any kind
- **Zero-auth design**: the agent's permissions are exactly the workflow's `permissions:` declaration, enforced by GitHub
- **Model-agnostic**: Anthropic / OpenAI / Google / any OpenAI-compatible gateway — switch models without touching the prompt
- **Ecosystem passthrough**: AGENTS.md, skills, and any MCP server already in the repo are consumed directly — no invented config format

## Quick start

Three steps to give your repo an issue-triage agent:

1. Create `.github/workflows/triage.yml`:

```yaml
name: Issue Triage
on:
  issues:
    types: [opened]

permissions:
  contents: read
  issues: write

jobs:
  its:
    runs-on: ubuntu-latest
    steps:
      - uses: minorcell/its@v1
        with:
          prompt: |
            A new issue is attached below. Classify it (bug / feature request /
            question) and apply the matching label; comment asking for repro
            steps when missing; answer common questions directly.
        env:
          ITS_PROTOCOL: anthropic-messages
          ITS_MODEL: claude-sonnet-5
          ITS_API_KEY: ${{ secrets.ITS_API_KEY }}
```

2. In Settings → Secrets and variables → Actions, add a secret `ITS_API_KEY` with your model provider's API key. To manage `ITS_PROTOCOL` and `ITS_MODEL` org-wide, replace the literals with `${{ vars.XXX }}` references.

3. Open a new issue. ITS completes a run within minutes: classify, label, comment when needed. The whole process is visible live on the Actions tab.

## Input reference

`with:` parameters:

| Parameter | Required | Description                                                                                                                                                         |
| --------- | -------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `prompt`  | ✅       | Natural-language description of what this run should accomplish. Event context (new issue body, PR diff, actor, etc.) is appended automatically — no parsing needed |
| `tools`   |          | Additional tools, array form                                                                                                                                        |
| `skills`  |          | Skills, array form. `github.com/<owner>/<repo>` is a remote skill repo installed via skills.sh; `.` enables this repo's `.agents/skills/` directory                 |
| `mcp`     |          | MCP server launch commands, array form. Credentials reference `$VAR` slots, resolved from `env:`                                                                    |

`env:` variables:

| Variable       | Required | Description                                                                                                                                   |
| -------------- | -------- | --------------------------------------------------------------------------------------------------------------------------------------------- |
| `ITS_PROTOCOL` | ✅       | Protocol name (not vendor name): `anthropic-messages` \| `openai-chat-completions` \| `openai-responses` \| `gemini`                          |
| `ITS_MODEL`    | ✅       | Model ID, e.g. `claude-sonnet-5`, `gpt-5`, `gemini-2.5-pro`                                                                                   |
| `ITS_API_KEY`  | ✅       | API key for the current protocol                                                                                                              |
| `ITS_BASE_URL` |          | Base URL when using a third-party gateway; `ITS_PROTOCOL` should name the protocol the gateway implements (usually `openai-chat-completions`) |

Connection info (protocol, model, key, gateway URL) lives together in `env:`, grouped with secrets, so it can be managed org-wide via `${{ vars.XXX }}`; `with:` describes behavior only.

`ITS_PROTOCOL` values are protocol names, not vendor names: `openai-chat-completions` covers every OpenAI-compatible gateway (DeepSeek, Groq, OpenRouter, Ollama, etc.), with the gateway URL set via `ITS_BASE_URL`; `openai-responses` is OpenAI's Responses API — a different wire protocol from Chat Completions, not interchangeable; `anthropic-messages` likewise covers Anthropic-compatible gateways; `gemini` maps to the Gemini API.

## More examples

### PR review

To read code, check out explicitly:

```yaml
name: PR Review
on:
  pull_request:
    types: [opened, synchronize]

permissions:
  contents: read
  pull-requests: write

jobs:
  its:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: minorcell/its@v1
        with:
          prompt: |
            The working directory has the current PR checked out. Post inline
            review comments in the order correctness > security >
            maintainability, then a final summary comment.
        env:
          ITS_PROTOCOL: openai-chat-completions
          ITS_MODEL: gpt-5
          ITS_API_KEY: ${{ secrets.ITS_API_KEY }}
```

### Scheduled weekly digest

```yaml
name: Weekly Digest
on:
  schedule:
    - cron: '0 9 * * 1' # every Monday 9:00

permissions:
  contents: read
  discussions: write

jobs:
  its:
    runs-on: ubuntu-latest
    steps:
      - uses: minorcell/its@v1
        with:
          prompt: |
            Summarize the last 7 days of commits, PRs, and issues into a weekly
            digest posted to Discussions, organized as progress / problems /
            next week.
        env:
          ITS_PROTOCOL: gemini
          ITS_MODEL: gemini-2.5-pro
          ITS_API_KEY: ${{ secrets.ITS_API_KEY }}
```

### Third-party gateway (OpenAI-compatible)

For any OpenAI-compatible third-party gateway (DeepSeek, private proxies, etc.), set `ITS_PROTOCOL` to the protocol it implements and point `ITS_BASE_URL` at it:

```yaml
- uses: minorcell/its@v1
  with:
    prompt: |
      Read this repo's README and the latest 20 commits,
      then fill in missing documentation sections.
  env:
    ITS_PROTOCOL: openai-chat-completions
    ITS_MODEL: deepseek-chat
    ITS_API_KEY: ${{ secrets.DEEPSEEK_API_KEY }}
    ITS_BASE_URL: https://api.deepseek.com/v1
```

### Skills: local and remote

```yaml
- uses: minorcell/its@v1
  with:
    prompt: |
      Prepare a release using the repo's release-flow skill:
      check the changelog, bump versions, open a release PR.
    skills:
      - . # all skills under this repo's .agents/skills/
      - github.com/minorcell/skills # remote skill repo, installed via skills.sh
  env:
    ITS_PROTOCOL: anthropic-messages
    ITS_MODEL: claude-sonnet-5
    ITS_API_KEY: ${{ secrets.ITS_API_KEY }}
```

### MCP server with credentials

For remote MCP calls that need credentials, keep secrets out of `with:` — reference `$VAR` slots resolved from `env:`:

```yaml
- uses: minorcell/its@v1
  with:
    prompt: |
      A new issue is attached below. Query the internal stats tool
      and reply to the user with data, then apply the matching label.
    mcp:
      - 'npx -y @team/stats-server --token $MCP_TOKEN'
  env:
    ITS_PROTOCOL: openai-chat-completions
    ITS_MODEL: gpt-5
    ITS_API_KEY: ${{ secrets.ITS_API_KEY }}
    MCP_TOKEN: ${{ secrets.MCP_TOKEN }}
```

## Repo ecosystem

ITS invents no config format of its own — it consumes what is already in the repo:

- **AGENTS.md**: loaded automatically when present at the repo root, serving as the agent's repository context (the same way Claude Code reads CLAUDE.md).
- **`.agents/skills/`**: this repo's skills directory; enable with `skills: ['.']`.
- **Remote skill repos**: declare `github.com/<owner>/<repo>`; installed via skills.sh.
- **MCP servers**: declared as command arrays; servers from any ecosystem can be plugged in.

## Permission model

ITS introduces no permission system of its own. What the agent can do is determined by the `permissions:` declaration in the workflow file; the toolset expands and contracts accordingly:

- Without `issues: write`, no "post a comment" tool exists in the agent's toolbox;
- When triggered by a fork PR, GitHub automatically downgrades the token to read-only, and the agent is left with read tools only;
- The API key lives only in your secrets — ITS never touches or resells it.

This boundary is enforced by GitHub and holds regardless of ITS.

## Caveats

- **Async execution**: minutes of latency between trigger and completion (queueing + environment startup). ITS suits asynchronous tasks, not real-time conversations.
- **Cost**: every run consumes tokens from your model account; scope the prompt explicitly.
- **Idempotency**: on duplicate triggers of the same event, ITS checks whether it already handled it, to avoid repeated replies.
- **Auditability**: every step (tool calls, model output) is in the Actions logs, and the artifacts are ordinary GitHub objects.
- **Prompt injection**: event content such as issue bodies is treated as untrusted data and handled as data only.

## Roadmap

- **Phase 1**: any GitHub user, with one workflow file and one secret, gets ITS to accomplish a real task in their repo without ever leaving GitHub.
- **Phase 2**: users never learn a new format for ITS — existing AGENTS.md, skills, and any MCP server are consumed directly; switching models or protocols never changes the prompt.
- **Phase 3**: a team composes multiple ITS runs into pipelines (plan, execute, review, each with its own role) — what gets reused is not code but other people's workflow files.
- **Phase 4 (to validate)**: enterprise environments (self-hosted runners, private gateways, private models) with an experience identical to the public internet.

## Status

🚧 In development, interfaces may change.
