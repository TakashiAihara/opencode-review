# opencode-review

A reusable GitHub Actions workflow that reviews each pull request with [opencode](https://github.com/anomalyco/opencode) and posts the review as one comment in Japanese.

- The model is `opencode/glm-5.3-flash-free` from opencode Zen's free tier, which opencode's docs list as not used for training and kept 0 days. Zen's free tier only answers requests from opencode itself, which is why this runs opencode rather than calling the model directly.
- The agent can only read: it runs as opencode's `plan` agent with no shell, no file edits, no web fetch or search, and no paths outside the repository except the PR's diff, which a step writes beforehand. The repository's own opencode config (`opencode.json`, `.opencode/`), its `.claude/skills` and `.agents/skills`, and `CLAUDE.md` are not loaded, so a PR cannot widen these. opencode's database, log and global config live in directories of the run's own.
- Drafts are reviewed too. The latest of a burst of pushes gets a new review (earlier runs are cancelled); an earlier comment is not edited.
- Pull requests from forks, and runs started by a bot, are skipped: forks and Dependabot get no secrets, and opencode refuses bot actors.
- opencode's session (the prompt, the PR text and files the model read, its replies, tool calls and reasoning, and subagent sessions) and its log are kept for 14 days as the run's artifact `opencode-review-logs`, for tuning the prompt. In a public repository anyone signed in to GitHub can download it. If the bundle contains the API key or the GitHub token as plain text, or cannot be checked, it is deleted and the job fails.

## Use

1. Add an opencode Zen API key as the repository secret `OPENCODE_API_KEY`. Keep its workspace at no balance and no auto-recharge: a leaked key then reaches the free models only. Paid models need a balance, and with it a leaked key spends it.
2. Add `.github/workflows/opencode-review.yml`:

```yaml
name: opencode-review

on:
  pull_request:
    types: [opened, synchronize, reopened]

permissions:
  contents: read

jobs:
  opencode:
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

`opencode-version` picks the [opencode release](https://github.com/anomalyco/opencode/releases) to review with, as `1.Y.Z` (2.x has no `github run` command). It defaults to `1.19.5`, and a new upstream release does not change it. Only an immutable release is accepted, and the download is checked against the sha256 digest GitHub recorded for it. opencode's own GitHub action is not used, because it takes no version and installs whatever release is marked latest.

## Review instructions per path

To give the reviewer guidance for some files, add `.github/opencode-review.yaml`. It takes the same shape as CodeRabbit's [`reviews.path_instructions`](https://docs.coderabbit.ai/reference/configuration), so entries can be copied from a `.coderabbit.yaml` as long as their globs avoid the syntax listed below:

```yaml
reviews:
  path_instructions:
    - path: "**/*.ts"
      instructions: |
        Point out a try/catch that swallows an error.
    - path: ".github/workflows/**"
      instructions: |
        Check that third-party actions are pinned to a commit SHA.
```

- Only entries whose `path` glob matches a file changed by the pull request are passed to the reviewer, with the matched files listed. For those files they decide what the reviewer checks over the general review prompt, so an entry can ask for checks the prompt leaves out, such as style. The output format and the read-only rules stay.
- Globs are matched by git's `:(glob)` pathspec, not CodeRabbit's minimatch. `**/*.ts` matches `a.ts` at the root too, and `src` matches everything under `src/`. Braces (`**/*.{ts,js}`), extglob (`+(a|b)`), and a leading `!` or `#` are not supported and fail the job; split or rewrite such an entry. A `{` is refused even inside `[...]`.
- The file is read from the base branch, not the pull request, so a pull request cannot edit it to change its own review. An edit to the file takes effect once it is merged.
- The job fails when the file has no `reviews.path_instructions` list (a misspelt `reviews` or `path_instructions`, or `path_instructions` at the top level), when an entry lacks `path` or `instructions` or has instructions over 20000 characters, and when the file is not valid YAML. Other keys are ignored, so a misspelt key next to the list is not caught. The run log says how many entries were read and which matched.
- This is separate from `AGENTS.md`, which the reviewer does not load at the repository root. An `AGENTS.md` or `CONTEXT.md` in a subdirectory, including one the pull request adds, is still attached when the reviewer reads a file there; opencode has no setting to stop that. `CLAUDE.md` is not loaded.


## Limits

- Public repositories only for now. opencode fetches the PR branch without credentials.
- When a free model is withdrawn, the job fails until `model` is changed here.
- The model reads the PR's diff and messages, so text in them can steer what the review says. With no shell and no edits, the comment is the only thing it writes; the session that opencode records goes into the artifact.
