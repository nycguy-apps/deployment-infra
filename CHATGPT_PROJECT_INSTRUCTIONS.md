# Standard deployment instructions for ChatGPT-built apps

Use these instructions for new applications created under the `nycguy-apps` GitHub organization.

1. Treat GitHub as the source of truth. Commit application code, tests, deployment configuration, and documentation.
2. Use Cloudflare Workers with static assets for production hosting unless the application has a documented reason to use another platform.
3. Keep Cloudflare configuration in `wrangler.jsonc`. Do not require routine manual Cloudflare dashboard edits.
4. Add `.github/workflows/deploy.yml` that calls `nycguy-apps/deployment-infra/.github/workflows/deploy-cloudflare.yml@main`.
5. Use `secrets: inherit`. Never request, print, log, retrieve, or commit the values of `CLOUDFLARE_API_TOKEN` or `CLOUDFLARE_ACCOUNT_ID`.
6. Deploy production from `main` only after the repository's validation commands succeed.
7. Add a public health endpoint for Worker/API apps when practical and pass its URL to the reusable workflow.
8. Prefer free/open services and free tiers. Do not add paid infrastructure without explicit approval.
9. After deployment, verify production and report the commit SHA and deployment/test status.
10. Preserve existing production URLs/resources when continuing an established application unless migration is explicitly requested.

Minimal caller workflow:

```yaml
name: Verify and deploy

on:
  push:
    branches: [main]
  workflow_dispatch:

permissions:
  contents: read

jobs:
  deploy:
    uses: nycguy-apps/deployment-infra/.github/workflows/deploy-cloudflare.yml@main
    secrets: inherit
```
