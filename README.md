# gh-workflows

A central repository for reusable GitHub Actions workflows.

## Available Workflows

### AI PR Review

AI-powered code review using [PR-Agent](https://github.com/qodo-ai/pr-agent) with automatic project context loading. Reads your `CLAUDE.md` and `AGENTS.md` files so the reviewer understands your codebase conventions.

**Features:**
- Automatic PR description, code review, and improvement suggestions
- Loads `CLAUDE.md` and `AGENTS.md` as review context automatically
- Committable inline suggestions (one-click accept on GitHub)
- Configurable model — defaults to DeepSeek V3 on Fireworks (~$0.02/PR)
- Works with any LiteLLM-supported provider (Fireworks, Anthropic, OpenAI, etc.)

### Lighthouse CI

Runs Lighthouse audits against Vercel preview deployments and posts results as a PR comment.

**Features:**
- Runs Lighthouse against preview deployment URLs
- Posts formatted results as PR comment
- Scores with visual indicators:
  - 🟢 Score >= 90
  - 🟠 Score >= 50
  - 🔴 Score < 50
- Reports on Performance, Accessibility, Best Practices, and SEO
- Shows detailed diagnostics for any score below 85%

## Usage

### AI PR Review

Create `.github/workflows/pr-review.yml` in your repository:

```yaml
name: AI PR Review

on:
  pull_request:
    types: [opened, reopened, ready_for_review, synchronize]

jobs:
  review:
    uses: queso/gh-workflows/.github/workflows/pr-review.yml@main
    permissions:
      contents: read
      pull-requests: write
    secrets:
      llm_api_key: ${{ secrets.FIREWORKS_AI_API_KEY }}
```

That's it. Every PR gets an auto-generated description, code review, and improvement suggestions — all informed by your project's `CLAUDE.md`.

#### With Anthropic Claude

```yaml
jobs:
  review:
    uses: queso/gh-workflows/.github/workflows/pr-review.yml@main
    permissions:
      contents: read
      pull-requests: write
    with:
      model: 'anthropic/claude-sonnet-4-6'
      fallback_model: 'anthropic/claude-haiku-4-5-20251001'
    secrets:
      llm_api_key: ${{ secrets.ANTHROPIC_KEY }}
```

#### With custom context and instructions

```yaml
jobs:
  review:
    uses: queso/gh-workflows/.github/workflows/pr-review.yml@main
    permissions:
      contents: read
      pull-requests: write
    with:
      context_file: 'CLAUDE.md'
      extra_instructions: 'This project uses Next.js App Router. Prefer server components.'
      auto_describe: false  # skip auto-description if you write your own
    secrets:
      llm_api_key: ${{ secrets.FIREWORKS_AI_API_KEY }}
```

#### Options

| Input | Default | Description |
|-------|---------|-------------|
| `model` | `fireworks_ai/.../deepseek-v3p2` | LiteLLM model identifier |
| `fallback_model` | `fireworks_ai/.../deepseek-v3p1` | Fallback if primary fails |
| `context_file` | `CLAUDE.md` | Project context file path |
| `extra_instructions` | `''` | Additional review instructions |
| `auto_review` | `true` | Run /review on PR open |
| `auto_describe` | `true` | Run /describe on PR open |
| `auto_improve` | `true` | Run /improve on PR open |
| `commitable_suggestions` | `true` | Inline suggestions you can accept with one click |

### Lighthouse CI for Vercel Previews

Create `.github/workflows/lighthouse.yml` in your repository:

```yaml
name: Lighthouse CI

on:
  deployment_status:

jobs:
  lighthouse:
    if: github.event.deployment_status.state == 'success' && github.event.deployment.environment == 'Preview'
    uses: queso/gh-workflows/.github/workflows/lighthouse.yml@main
    permissions:
      contents: read
      pull-requests: write
      deployments: read
    with:
      target_url: ${{ github.event.deployment_status.target_url }}
      deployment_id: ${{ github.event.deployment.id }}
```

## Required Permissions

Both workflows need:

```yaml
permissions:
  contents: read        # Checkout the repo
  pull-requests: write  # Post comments to PR
```

Lighthouse CI additionally needs:

```yaml
permissions:
  deployments: read     # Look up deployment to find associated PR
```

## License

MIT
