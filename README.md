# nycguy-apps deployment infrastructure

Shared deployment automation for ChatGPT-built applications in the **nycguy-apps** GitHub organization.

## What this solves

New public app repositories can inherit the organization's Cloudflare credentials and use one standard deployment workflow. The app repo contains its code and `wrangler.jsonc`; this repository contains the common CI/CD logic.

```text
ChatGPT
   |
   v
GitHub app repo
   |
   | push to main
   v
nycguy-apps/deployment-infra
   |
   +-- lint / type-check / tests
   +-- production build
   +-- optional smoke test
   +-- Wrangler deployment
   +-- optional production health check
   |
   v
Cloudflare Workers + static assets
```

## Organization secrets

The organization supplies:

- `CLOUDFLARE_API_TOKEN`
- `CLOUDFLARE_ACCOUNT_ID`

Applications should consume them with `secrets: inherit`. Never print, retrieve, copy into source, or commit either value.

## New app: minimal deployment workflow

Create `.github/workflows/deploy.yml` in the app repository:

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

Then add a `wrangler.jsonc`. A standard Vite/static app can start with:

```jsonc
{
  "$schema": "node_modules/wrangler/config-schema.json",
  "name": "my-app",
  "compatibility_date": "2026-09-28",
  "assets": {
    "directory": "./dist",
    "not_found_handling": "single-page-application"
  }
}
```

On a push to `main`, the reusable workflow:

1. installs dependencies
2. runs lint when present
3. runs type-checking when present
4. runs tests when present
5. builds production assets
6. optionally runs an integration/smoke command
7. deploys with Wrangler
8. optionally verifies a production health URL

## Optional health check

```yaml
jobs:
  deploy:
    uses: nycguy-apps/deployment-infra/.github/workflows/deploy-cloudflare.yml@main
    with:
      healthcheck_url: https://my-app.example.workers.dev/health
    secrets: inherit
```

The health check retries transient failures and fails the Actions run if production remains unreachable.

## Command overrides

Apps can override any standard command:

```yaml
with:
  install_command: npm ci
  lint_command: npm run lint
  typecheck_command: npm run typecheck
  test_command: npm run test -- --run
  build_command: npm run build
  smoke_command: npm run test:live
  deploy_command: npx --yes wrangler@4 deploy
```

Pass an empty string to skip an optional command.

## Validation

Two workflows validate this infrastructure without creating Cloudflare resources:

- **Validate Cloudflare organization secrets** checks that the organization credentials are present and runs `wrangler whoami`.
- **Test reusable deployment workflow** invokes the shared workflow exactly as an app repo would, but replaces deployment with `wrangler whoami`.

## Standard instructions for ChatGPT projects

See [CHATGPT_PROJECT_INSTRUCTIONS.md](./CHATGPT_PROJECT_INSTRUCTIONS.md). Those instructions are designed to be copied into future ChatGPT app projects so they automatically use this deployment platform.

## Security

- Use a scoped Cloudflare API token, not the Global API Key.
- Cloudflare credentials stay in GitHub Actions secrets.
- Do not put credentials in `.env`, Wrangler configuration, issues, commits, logs, or ChatGPT prompts.
- Production deployment is from `main`.
- Do not weaken tests simply to make a deployment pass.
- Do not create replacement Cloudflare resources for established apps unless a migration is explicitly intended.

## Free-tier design

This infrastructure is intentionally designed around free services. The current organization-secret approach is intended for eligible **public repositories** on GitHub Free. Before making an app repository private, verify the private-repository secret and hosting strategy so an existing production deployment is not interrupted.
