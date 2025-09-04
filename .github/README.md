# Single App GitHub Actions

This repository contains reusable GitHub Actions for deploying and managing single applications to Cloudflare Workers.

## Actions Overview

### Core Actions

- **`deploy-app`**: Deploy a single app to Cloudflare Workers
- **`lighthouse`**: Run Lighthouse performance tests on deployed apps
- **`ci-checks`**: Run TypeScript checks, linting, and security scans
- **`upsert-preview-comment`**: Create or update PR preview comments

### Automatic Workflows

- **`pr-preview.yml`**: PR preview deployments with CI checks
- **`deploy-dev.yml`**: Automatic deployment to dev on push to main
- **`deploy-production.yml`**: Automatic deployment to production on releases
- **`rollback.yml`**: Manual rollback workflow

## Usage in Consuming Repositories

### 1. Copy Workflow Files

Copy the workflow files from `apps-v3/workflows/` to your repository's `.github/workflows/` directory:

```bash
# Copy all workflows
cp apps-v3/workflows/*.yml /path/to/your-repo/.github/workflows/

# Or copy specific workflows
cp apps-v3/workflows/pr-preview.yml /path/to/your-repo/.github/workflows/
cp apps-v3/workflows/deploy-dev.yml /path/to/your-repo/.github/workflows/
cp apps-v3/workflows/deploy-production.yml /path/to/your-repo/.github/workflows/
cp apps-v3/workflows/rollback.yml /path/to/your-repo/.github/workflows/
```

### 2. Update App Name

In each workflow file, update the `app-name` parameter to match your app:

```yaml
- name: Deploy to Preview
  uses: centrifuge/github-actions-lib/apps-v3/actions/deploy-app@main
  with:
    environment: dev
    build-mode: testnet
    app-name: 'your-app-name'  # Update this
```

### 3. Configure Secrets

Set these secrets in your repository:

- **`CLOUDFLARE_API_TOKEN`**: Cloudflare API token with Workers permissions
- **`CLOUDFLARE_ACCOUNT_ID`**: Your Cloudflare account ID


### 4. App Requirements

Your app should have:

1. **`package.json`** with a `build` script
2. **`wrangler.toml`** configured for your environments
3. **`pnpm`** as package manager (pnpm-lock.yaml)

## Workflow Behavior

### PR Preview (`pr-preview.yml`)
- **Triggers**: Pull request events
- **Actions**: 
  - Runs CI checks (TypeScript, linting, security)
  - Deploys to dev environment
  - Runs Lighthouse performance tests
  - Creates/updates PR preview comment

### Development Deployment (`deploy-dev.yml`)
- **Triggers**: Push to main branch
- **Actions**:
  - Runs CI checks
  - Deploys to dev environment
  - Runs Lighthouse performance tests

### Production Deployment (`deploy-production.yml`)
- **Triggers**: GitHub releases
- **Actions**:
  - Downloads build artifacts from the release
  - Deploys to production (released) or staging (prereleased)

### Manual Rollback (`rollback.yml`)
- **Triggers**: Manual workflow dispatch
- **Actions**:
  - Downloads build artifacts from specified release
  - Deploys to specified environment using `deploy-app` with `release-tag`

## Environment Configuration

### Development (dev)
- Uses `wrangler deploy --env dev`
- Builds with `--mode testnet`

### Staging
- Uses `wrangler versions upload --env prod --preview-alias staging`
- Builds with `--mode mainnet`
- Creates a preview deployment

### Production
- Uses `wrangler deploy --env prod`
- Builds with `--mode mainnet`
- Deploys to production domain

## Using Actions Directly

You can also use these actions directly in your own workflows:

```yaml
- name: Deploy to Dev
  uses: centrifuge/github-actions-lib/apps-v3/actions/deploy-app@main
  with:
    environment: dev
    build-mode: testnet

- name: Lighthouse Performance Test
  uses: centrifuge/github-actions-lib/apps-v3/actions/lighthouse@main
  with:
    url: ${{ steps.deploy.outputs.deployment-url }}
    app-name: 'my-app'
```

## Benefits

- **Centralized**: All deployment logic in one place
- **Consistent**: Same deployment process across all apps
- **Maintainable**: Changes to deployment logic only need to be made once
- **Minimal**: Consuming repositories only need thin workflow files
- **Automatic**: Full CI/CD pipeline with PR previews and production releases

## Troubleshooting

### Common Issues

1. **Missing wrangler.toml**: Action will skip deployment and show a warning
2. **Build failures**: Check your package.json build script
3. **Deployment failures**: Verify Cloudflare secrets and permissions
4. **Smoke test failures**: Check if your app is responding on the deployed URL

### Debug Mode

Add `ACTIONS_STEP_DEBUG: true` to your repository secrets to enable debug logging.
