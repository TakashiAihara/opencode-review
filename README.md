# opencode-review

A reusable GitHub Actions workflow that reviews each pull request with [opencode](https://github.com/anomalyco/opencode) and posts the review as one comment in Japanese.

- The model is `opencode/glm-5.3-flash-free` from opencode Zen's free tier, which opencode's docs list as not used for training and kept 0 days. Zen's free tier only answers requests from opencode itself, which is why this runs opencode rather than calling the model directly.
- The agent can only read: it runs as opencode's `plan` agent, file edits, web fetch and web search are denied, and the shell is limited to `git diff`, `git log` and `git show`.
- Drafts are reviewed too. Each push to the PR gets a new review; an earlier comment is not edited.
- Pull requests from forks, and runs started by Dependabot, are skipped: neither gets the repository's secrets.

## Use

1. Add an opencode Zen API key as the repository secret `OPENCODE_API_KEY`. A workspace with no balance and no auto-recharge keeps a leaked key to the free models.
2. Add `.github/workflows/opencode-review.yml`:

```yaml
name: opencode-review

on:
  pull_request:
    types: [opened, synchronize, reopened]

permissions:
  contents: read

jobs:
  review:
    permissions:
      contents: read
      pull-requests: write
      issues: write
    uses: TakashiAihara/opencode-review/.github/workflows/review.yml@main
    secrets:
      OPENCODE_API_KEY: ${{ secrets.OPENCODE_API_KEY }}
```

`@main` follows this repository, so a model or prompt change here reaches every caller at once. Pin a commit instead if you want to review each change before it runs.

To use another model, pass `with: { model: <provider/id> }`.

## Limits

- Public repositories only for now. opencode fetches the PR branch without credentials, and the opencode binary the action installs is the latest release, not a pinned one.
- When a free model is withdrawn, the job fails until `model` is changed here.
