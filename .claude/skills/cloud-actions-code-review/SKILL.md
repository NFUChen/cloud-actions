---
name: cloud-actions-code-review
description: Generate GitHub Actions caller workflows that use Claude Code to review pull request diffs and post inline Critical/High severity findings. Use this skill whenever the user mentions AI code review, Claude PR review, automated code review, inline review comments, or wants CI to review pull requests automatically.
---

# Claude Code Review Workflow

Generate caller workflow YAML for the `NFUChen/cloud-actions` Claude Code review reusable workflow. Uses Claude Code to review the PR diff for correctness, security, and reliability problems, then posts inline comments for Critical/High findings.

## Base URL

```
NFUChen/cloud-actions/.github/workflows/claude-code-review.yml@main
```

## Inputs

| Name | Type | Required | Default | Description |
|------|------|----------|---------|-------------|
| `pr-number` | string | no | PR from event | Pull request number; for `workflow_dispatch` use |
| `additional-prompt` | string | no | `""` | Extra instructions appended to the review prompt |
| `allowed-tools` | string | no | `""` | Comma-prefixed list of additional allowed tools, e.g. `,Bash(npm:*),Bash(pipenv:*)` |
| `model` | string | no | `claude-sonnet-4-5` | Claude model ID |
| `anthropic-base-url` | string | no | `""` | Optional Anthropic-compatible API base URL (e.g. proxy or gateway) |

## Secrets

| Name | Required | Description |
|------|----------|-------------|
| `anthropic-api-key` | yes | Anthropic API key |

## Caller Example

```yaml
name: Claude Code Review

on:
  pull_request:
    types: [opened, reopened, synchronize]

jobs:
  review:
    uses: NFUChen/cloud-actions/.github/workflows/claude-code-review.yml@main
    secrets:
      anthropic-api-key: ${{ secrets.ANTHROPIC_API_KEY }}
```

## Common Patterns

### Extra Review Instructions

```yaml
jobs:
  review:
    uses: NFUChen/cloud-actions/.github/workflows/claude-code-review.yml@main
    with:
      additional-prompt: "Pay extra attention to SQL query construction and pagination logic."
    secrets:
      anthropic-api-key: ${{ secrets.ANTHROPIC_API_KEY }}
```

### Allow Extra Tools (e.g. run tests or linters)

```yaml
jobs:
  review:
    uses: NFUChen/cloud-actions/.github/workflows/claude-code-review.yml@main
    with:
      allowed-tools: ",Bash(npm:*),Bash(pipenv:*)"
    secrets:
      anthropic-api-key: ${{ secrets.ANTHROPIC_API_KEY }}
```

### Specify Model

```yaml
jobs:
  review:
    uses: NFUChen/cloud-actions/.github/workflows/claude-code-review.yml@main
    with:
      model: "claude-sonnet-4-5"
    secrets:
      anthropic-api-key: ${{ secrets.ANTHROPIC_API_KEY }}
```

### Custom Anthropic Base URL

Route requests through a proxy or an Anthropic-compatible gateway:

```yaml
jobs:
  review:
    uses: NFUChen/cloud-actions/.github/workflows/claude-code-review.yml@main
    with:
      anthropic-base-url: "https://my-gateway.example.com"
    secrets:
      anthropic-api-key: ${{ secrets.ANTHROPIC_API_KEY }}
```

### Manual Trigger via workflow_dispatch

```yaml
name: Claude Code Review (manual)

on:
  workflow_dispatch:
    inputs:
      pr-number:
        description: "PR number to review"
        required: true
        type: string

jobs:
  review:
    uses: NFUChen/cloud-actions/.github/workflows/claude-code-review.yml@main
    with:
      pr-number: ${{ inputs.pr-number }}
    secrets:
      anthropic-api-key: ${{ secrets.ANTHROPIC_API_KEY }}
```

## Generation Guidelines

When generating caller workflows:

- Default trigger should be `pull_request: types: [opened, reopened, synchronize]`
- Remind the user to store their Anthropic API key as `ANTHROPIC_API_KEY` in repo secrets
- Only include `additional-prompt` if the user wants extra focus areas or instructions appended to the review
- Only include `allowed-tools` if the user wants Claude to run additional commands (tests, linters, package managers) during review
- Only include `model` if the user wants a different model than the default
- Only include `anthropic-base-url` if the user mentions a proxy, gateway, or self-hosted Anthropic-compatible endpoint
- For `workflow_dispatch` callers, include the `pr-number` input mapping
- The workflow only reports Critical and High severity findings (security, correctness, reliability) via inline PR comments; it does not comment on style or minor issues
- Note that `pull_request` runs from forked repos do not receive secrets, so the workflow only works for same-repo PRs by default
