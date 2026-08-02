# gh-workflows

A central repository for reusable GitHub Actions workflows.

## Available Workflows

### AI PR Review

AI-powered code review using [PR-Agent](https://github.com/qodo-ai/pr-agent) with automatic project context loading. Reads your `CLAUDE.md` and `AGENTS.md` files so the reviewer understands your codebase conventions.

**Features:**
- Automatic PR description, code review, and improvement suggestions
- Loads `CLAUDE.md` and `AGENTS.md` as review context automatically
- Committable inline suggestions (one-click accept on GitHub)
- Auto-selects the best model by PR size (Qwen3 Coder → Kimi K2.6 → DeepSeek V4 Pro)
- Works with any LiteLLM-supported provider (Fireworks, Anthropic, OpenAI, etc.)

### Nitpick PR Review

Multi-station PR review on the Conduit kernel. Classifies the diff, fans out parallel domain reviewers (security / correctness / performance / conventions / tests) across coupling-aware shards, then a frontier coordinator dedups, assigns severity, and posts **one** batched review with inline anchors.

**Features:**
- Parallel specialist reviewers instead of one generalist pass — each stays in its lane
- Shard planner slices large diffs so per-prompt size stays bounded (no diff cap, no coverage hole)
- Coordinator dedups overlapping findings and gates noise before anything is posted
- Reads `CLAUDE.md` and `AGENTS.md` for project conventions
- Post-mortem + model-spend reports written to the job summary on every run

**Requires** access to the private Nitpick flow image — see the access note in the usage section below.

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
| `model` | Auto-select by PR size | LiteLLM model identifier (overrides auto-select) |
| `fallback_model` | `fireworks_ai/.../kimi-k2p6` | Fallback if primary fails |
| `context_file` | `CLAUDE.md` | Project context file path |
| `extra_instructions` | `''` | Additional review instructions |
| `auto_review` | `true` | Run /review on PR open |
| `auto_describe` | `true` | Run /describe on PR open |
| `auto_improve` | `true` | Run /improve on PR open |
| `commitable_suggestions` | `true` | Inline suggestions you can accept with one click |

### Nitpick PR Review

Create `.github/workflows/pr-review.yml` in your repository:

```yaml
name: AI PR Review

on:
  pull_request:
    types: [opened, reopened, ready_for_review, synchronize]

permissions:
  contents: read
  pull-requests: write
  packages: read

jobs:
  nitpick:
    # Fork PRs get no repo secrets — skip them rather than fail a check
    # the contributor can't fix. Only needed on public repositories.
    if: github.event.pull_request.head.repo.full_name == github.repository
    uses: queso/gh-workflows/.github/workflows/nitpick.yml@main
    permissions:
      contents: read
      pull-requests: write
      packages: read
    secrets:
      conduit_api_key: ${{ secrets.FIREWORKS_AI_API_KEY }}
      # Public repos only — see "Image access" below:
      # registry_token: ${{ secrets.GHCR_READ_PAT }}
```

The defaults point at Fireworks serverless for both the reviewers and the coordinator, so that one gateway key is the entire model configuration.

#### Image access

The flow image (`ghcr.io/queso/nitpick-flow`) is a **private** GHCR package, and how you authenticate depends on your repo's visibility:

| Consumer repo | What's required |
|---|---|
| **Private** | Grant the repo read access under the package's *Manage Actions access*. `GITHUB_TOKEN` then works — no `registry_token`. |
| **Public** | The package grant does not apply. Supply `registry_token` — a `read:packages` PAT stored as a repo secret. |

Note that GitHub does not allow a workflow in a public repository to call a reusable workflow stored in a private one. That's why this workflow lives here rather than beside the flow's own source.

#### Inputs

| Input | Default | Description |
|---|---|---|
| `flow_image` | `ghcr.io/queso/nitpick-flow:main` | Prebuilt engine + flow image |
| `reviewer_model` | `accounts/fireworks/models/deepseek-v4-flash-0731` | Cheap/code model for the specialist reviewers |
| `coordinator_model` | `accounts/fireworks/models/qwen3p7-plus` | Frontier model for the coordinator's judgment |
| `conduit_base_url` | `https://api.fireworks.ai/inference/v1` | OpenAI-compatible gateway. Front two providers with a litellm proxy and let the model strings route. |

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
