# 0loop

0loop is a GitHub Action that turns one workflow step into an agent run: a natural-language task, a model connection, repository context, and permission bounds.

- One step, same shape as `actions/checkout`
- Runs on your runner with your key and gateway; no SaaS, no GitHub App, no hosted service
- Models are described by wire protocol, model id, and gateway URL, not by vendor
- A run starts when the step starts and ends when the step ends

## Use

GitHub API access is set on the job. 0loop defaults to read-only tools.

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
          agent: |
            protocol: openai-responses
            model: your-model-id
            prompt: |
              Review this pull request. Report findings in the order
              correctness, security, and maintainability, then publish a summary.
        env:
          ZEROLOOP_BASE_URL: ${{ vars.ZEROLOOP_BASE_URL }}
          ZEROLOOP_API_KEY: ${{ secrets.ZEROLOOP_API_KEY }}
```

Set `ZEROLOOP_BASE_URL` and `ZEROLOOP_API_KEY` as repository or organization variables and secrets. The full API is in [the top-level API design](docs/plans/001-api-design/design.md).

## Status

0loop is a scaffold, not a working agent runner. The example is the target interface.

## Development

See [CONTRIBUTING.md](CONTRIBUTING.md).

## License

0loop is licensed under the [Apache License 2.0](LICENSE).
