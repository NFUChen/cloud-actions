---
name: cloud-actions-pr-description
description: Generate GitHub Actions caller workflows that use Claude Code to rewrite a PR description from the PR diff. Use this skill whenever the user mentions auto PR description, PR summary generation, Claude PR body, AI-generated pull request description, or wants CI to rewrite pull request descriptions automatically.
---

# PR Description Rewrite Workflow

Generate caller workflow YAML for the `NFUChen/cloud-actions` PR description rewrite reusable workflow. Uses Claude Code to read the PR diff and completely rewrite the PR description.

## Base URL

```
NFUChen/cloud-actions/.github/workflows/pr-description-rewrite.yml@main
```

## Inputs

| Name | Type | Required | Default | Description |
|------|------|----------|---------|-------------|
| `model` | string | no | `claude-sonnet-4-5` | Claude model ID |
| `max-diff-lines` | number | no | `3000` | Maximum diff lines included in the prompt |
| `extra-instructions` | string | no | `""` | Additional tone or formatting instructions |
| `anthropic-base-url` | string | no | `""` | Optional Anthropic-compatible API base URL (e.g. proxy or gateway) |
| `pr-number` | number | no | PR from event | Pull request number; for `workflow_dispatch` use |

## Secrets

| Name | Required | Description |
|------|----------|-------------|
| `anthropic-api-key` | yes | Anthropic API key |

## Caller Example

```yaml
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

## Common Patterns

### Custom Language or Tone

```yaml
jobs:
  rewrite:
    uses: NFUChen/cloud-actions/.github/workflows/pr-description-rewrite.yml@main
    with:
      extra-instructions: "Write in Traditional Chinese. Add a checklist section."
    secrets:
      anthropic-api-key: ${{ secrets.ANTHROPIC_API_KEY }}
```

### Specify Model

```yaml
jobs:
  rewrite:
    uses: NFUChen/cloud-actions/.github/workflows/pr-description-rewrite.yml@main
    with:
      model: "claude-sonnet-4-5"
    secrets:
      anthropic-api-key: ${{ secrets.ANTHROPIC_API_KEY }}
```

### Custom Anthropic Base URL

Route requests through a proxy or an Anthropic-compatible gateway:

```yaml
jobs:
  rewrite:
    uses: NFUChen/cloud-actions/.github/workflows/pr-description-rewrite.yml@main
    with:
      anthropic-base-url: "https://my-gateway.example.com"
    secrets:
      anthropic-api-key: ${{ secrets.ANTHROPIC_API_KEY }}
```

### Manual Trigger via workflow_dispatch

```yaml
name: Rewrite PR Description (manual)

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

## Generation Guidelines

When generating caller workflows:

- Default trigger should be `pull_request: types: [opened, reopened, synchronize]`
- Remind the user to store their Anthropic API key as `ANTHROPIC_API_KEY` in repo secrets
- Only include `extra-instructions` if the user specifies a language, tone, or extra sections
- Only include `model` if the user wants a different model than the default
- Only include `anthropic-base-url` if the user mentions a proxy, gateway, or self-hosted Anthropic-compatible endpoint
- For `workflow_dispatch` callers, include the `pr-number` input mapping
- Note that `pull_request` runs from forked repos do not receive secrets, so the workflow only works for same-repo PRs by default
