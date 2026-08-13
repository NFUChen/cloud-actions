# PR Description Rewrite

Reusable GitHub Actions workflow that uses Claude Code to automatically rewrite pull request descriptions based on the PR diff and commit messages.

## Prerequisites

- An Anthropic API key stored as a repository secret named `ANTHROPIC_API_KEY`
- Pull requests in your repository

## Usage

### Basic Setup

Create a workflow file in your repository:

```yaml
# .github/workflows/pr-description.yml
name: PR Description

on:
  pull_request:
    types: [opened, reopened, synchronize]

jobs:
  rewrite:
    uses: NFUChen/cloud-actions/.github/workflows/pr-description-rewrite.yml@main
    secrets:
      anthropic-api-key: ${{ secrets.ANTHROPIC_API_KEY }}
```

### Custom Language or Tone

```yaml
jobs:
  rewrite:
    uses: NFUChen/cloud-actions/.github/workflows/pr-description-rewrite.yml@main
    with:
      extra-instructions: "Write in Traditional Chinese. Include a testing checklist."
    secrets:
      anthropic-api-key: ${{ secrets.ANTHROPIC_API_KEY }}
```

### Use a Different Model

```yaml
jobs:
  rewrite:
    uses: NFUChen/cloud-actions/.github/workflows/pr-description-rewrite.yml@main
    with:
      model: "claude-opus-4"
    secrets:
      anthropic-api-key: ${{ secrets.ANTHROPIC_API_KEY }}
```

### Custom Anthropic Base URL

Route requests through a proxy or an Anthropic-compatible gateway. The value is passed to Claude Code as the `ANTHROPIC_BASE_URL` environment variable:

```yaml
jobs:
  rewrite:
    uses: NFUChen/cloud-actions/.github/workflows/pr-description-rewrite.yml@main
    with:
      anthropic-base-url: "https://my-gateway.example.com"
    secrets:
      anthropic-api-key: ${{ secrets.ANTHROPIC_API_KEY }}
```

Leave it unset to use the default Anthropic API endpoint.

### Manual Trigger

Allow manual PR description rewrites via `workflow_dispatch`:

```yaml
# .github/workflows/pr-description-manual.yml
name: Rewrite PR Description (Manual)

on:
  workflow_dispatch:
    inputs:
      pr-number:
        description: "PR number to rewrite"
        required: true
        type: number

jobs:
  rewrite:
    uses: NFUChen/cloud-actions/.github/workflows/pr-description-rewrite.yml@main
    with:
      pr-number: ${{ inputs.pr-number }}
    secrets:
      anthropic-api-key: ${{ secrets.ANTHROPIC_API_KEY }}
```

## Inputs

| Name | Type | Required | Default | Description |
|------|------|----------|---------|-------------|
| `model` | string | no | `claude-sonnet-4-5` | Claude model ID to use |
| `max-diff-lines` | number | no | `3000` | Maximum number of diff lines included in the prompt |
| `extra-instructions` | string | no | `""` | Additional tone or formatting instructions |
| `anthropic-base-url` | string | no | `""` | Anthropic-compatible API base URL; sets `ANTHROPIC_BASE_URL` |
| `pr-number` | number | no | Event PR | Pull request number; for `workflow_dispatch` use |

## Secrets

| Name | Required | Description |
|------|----------|-------------|
| `anthropic-api-key` | yes | Anthropic API key |

## How It Works

1. **Fetch PR context**: The workflow fetches the PR diff, title, and commit messages
2. **Truncate if needed**: If the diff exceeds `max-diff-lines`, it's truncated with a note
3. **Generate prompt**: A prompt is built with the diff, commit context, and any extra instructions
4. **Run Claude**: Claude Code analyzes the diff and generates a new description
5. **Update PR**: The PR description is completely rewritten with the generated content

## Generated Description Format

The generated description always includes:

- **Summary**: High-level overview of the PR
- **Changes**: Detailed list of changes
- **Testing**: Testing approach or checklist

## Disabling via Label (Extension Example)

The basic workflow runs on all PRs. To skip certain PRs, you can extend the workflow:

```yaml
on:
  pull_request:
    types: [opened, reopened, synchronize]

jobs:
  rewrite:
    if: ${{ !contains(github.event.pull_request.labels.*.name, 'skip-auto-description') }}
    uses: NFUChen/cloud-actions/.github/workflows/pr-description-rewrite.yml@main
    secrets:
      anthropic-api-key: ${{ secrets.ANTHROPIC_API_KEY }}
```

Add the label `skip-auto-description` to any PR to prevent automatic rewrites.

## Recommended Triggers

The workflow is designed to run when:
- A PR is **opened** (`opened`)
- A PR is **reopened** after being closed (`reopened`)
- New commits are pushed to the PR (`synchronize`)

This ensures the description stays in sync with the latest changes.

> Note: `pull_request` workflows from forked repositories do not receive repository secrets by default, and their `GITHUB_TOKEN` may not have permission to edit the PR. This workflow works for same-repository PRs by default.

## Setting Up Your API Key

1. Get an API key from https://console.anthropic.com/
2. Go to your repository **Settings** > **Secrets and variables** > **Actions**
3. Click **New repository secret**
4. Name: `ANTHROPIC_API_KEY`
5. Value: Your API key
6. Click **Add secret**

## Troubleshooting

### Empty Description Generated

If the workflow generates an empty description, it will keep the existing PR body and log a warning. This can happen if:
- The diff is too complex or large
- Claude cannot extract meaningful information

Check the job summary for diff statistics and whether truncation occurred.

### Diff Truncation

Large diffs are truncated to avoid context limits. If you see `Diff truncated: true` in the job summary:
- Increase `max-diff-lines` while watching for context-limit issues
- Split large PRs into smaller, focused changes
- Use `extra-instructions` to emphasize what matters most
