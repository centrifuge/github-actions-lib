## Workflows Overview

- **Two workflows** manage deployments:
  - **Dev (PR previews + push to main)**: `.github/workflows/deploy-dev.yml`
  - **Production (release-driven)**: `.github/workflows/deploy-production.yml`
- **Two Cloudflare builds per app**

### Dev: App PR Preview (`deploy-dev.yml`)
- **Triggers**:
  - Pushes to `main` touching `apps/**` or the workflow file
  - Pull requests touching `apps/**` or the workflow file
- **App selection (auto-detection)**:
  - Detects changed apps via `git diff`
  - If the workflow file itself changed, it runs **all apps** to test the workflow did not break anything
  - Only includes directories under `apps/*` that contain a `package.json`
- **Build and deploy**:
  - Reuses `.github/actions/build-app`
  - PRs: `wrangler versions upload --env dev` -> creates a preview under workers.dev domain
  - Main: `wrangler deploy --env dev` -> updates {app_name}-demo.k-f.dev
  - Skips deploy if `wrangler.toml` is missing

### Production: Deploy on Release (`deploy-production.yml`)
- **Triggers**: GitHub Releases (`released`, `prereleased`)
- **App selection**: Explicit matrix (e.g., `[derwa, launchpad]`)
  - Intentional for release control; adjust the matrix to include/exclude production apps
- **Build and deploy**:
  - Checks if a build artifact for the tag already exists; if yes, skips rebuilding
  - Reuses `.github/actions/build-app` with `build-mode: mainnet`
  - Uploads build artifact to the Release
  - `released`: `wrangler deploy --env prod` -> deploys to domain configured in `wrangler.toml`
  - `prereleased`: `wrangler versions upload --env prod --preview-alias staging` -> creates a preview of prod under a workers.dev domain which can be locked behind authentication if we want to.
  - Skips deploy if `wrangler.toml` is missing

### Rollback
- Retrieves the artifact from the release previously uploaded by the production job

### Reusable Build Action
- Path: `.github/actions/build-app`
- Inputs:
  - `app`: app directory name under `apps/`
  - `build-mode`: `testnet` (dev) or `mainnet` (prod)
- Expected to run something like `pnpm --filter <app> build --mode <build-mode>`. Skips `--mode` if it's not a Vite app (like functions)

### Adding a New App
1. Create your app under `apps/<your-app>/`
2. Ensure:
   - `package.json` with a `build` script
   - `wrangler.toml` configured for your environments (demo and prod)
3. For dev/PR previews: nothing else — it is auto-detected
4. For production releases: add the app to the `matrix.app` list in `.github/workflows/deploy-production.yml`

### Required secrets (one-time repo setup)
- **CLOUDFLARE_ACCOUNT_ID**: Your Cloudflare account ID.
- **CLOUDFLARE_API_TOKEN**: API token with at least:
  - **Account > Cloudflare Workers > Edit**
  - Plus any additional products your worker uses (e.g., KV, D1, R2) with Write permissions.

How to add:
- Go to GitHub → Settings → Secrets and variables → Actions → New repository secret.
- Create the two secrets above. These are shared by all apps and used by the deploy workflows.

App/runtime secrets (if your app needs them):
- Store application secrets in Cloudflare Worker environments, not in GitHub:
  - From the app directory: `wrangler secret put <NAME> --env dev`
  - And for prod: `wrangler secret put <NAME> --env prod`
- Alternatively set them in the Cloudflare dashboard under the worker’s Dev/Prod environments.

### Rationale and Notes
- Deployments consolidated into two workflows → fewer moving parts, easier debugging
- Auto-detection reduces boilerplate when adding new apps
- Explicit production matrix provides predictable release scope
- ToDo after final 1.0 release: production will release the app but will require a Cloudflare admin to redirect traffic to the new version

