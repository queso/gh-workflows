# gh-workflows

A central repository for reusable GitHub Actions workflows.

## Available Workflows

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

## Usage

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
    with:
      target_url: ${{ github.event.deployment_status.target_url }}
      deployment_id: ${{ github.event.deployment.id }}
```

### Required Permissions

The calling workflow needs these permissions for the reusable workflow to post PR comments:

```yaml
permissions:
  contents: read
  pull-requests: write
```

## License

MIT
