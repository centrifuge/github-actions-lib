# Centrifuge GitHub Actions Library

Centralized CI/CD pipelines for Centrifuge app repos (`apps-invest`,
`apps-management`), published as **reusable workflows** (`workflow_call`).
Each app keeps only thin caller workflows that pass app-specific inputs and
its own secrets — pipeline changes happen once, here, and roll out to every
consumer.

## Contents

```
.github/workflows/
  app-ci-checks.yml             PR quality gate: prettier, lint, codespell, audit, TruffleHog, pinact
  app-build-deploy-dev.yml      PR → preview deploy; push to main → demo deploy; optional Lighthouse
  app-build-deploy-release.yml  prereleased → staging (+ optional public-demo); released → production
  app-rollback.yml              manual rollback (prod: version traffic shift; staging: bundle re-upload)
  lib-ci.yml                    this repo's own CI (actionlint, pinact, yamllint)
actions/
  setup-app/                    Node + pnpm bootstrap (pnpm version from package.json "packageManager")
  build-app/                    build + artifact upload (+ release bundle upload on prerelease)
  deploy-app/                   wrangler deploy per environment (dev/demo/public-demo/staging/prod)
lighthouserc.json               shared LHCI config used by app-build-deploy-dev's performance job
```

## Security model — where secrets live

**This repo is public and holds no secrets.** Every reusable workflow declares
an explicit `secrets:` contract; callers pass their own repository secrets at
call time. `secrets: inherit` is never used — a workflow here can only ever
see what a caller explicitly hands it.

Per consumer repo you need:

| Kind | Name | Used by |
|---|---|---|
| Secret | `CLOUDFLARE_API_TOKEN` | all deploy/rollback workflows |
| Variable | `CLOUDFLARE_ACCOUNT_ID` | all deploy/rollback workflows (passed as an input) |
| Variable | `POOL_CACHE_BASE_URL` | apps-invest release builds only |
| GitHub Environments | `preview`, `demo`, `staging`, `production` (+ `public-demo` for apps-management) | deploy jobs |

Secrets must be **repository-level** (not environment-scoped): the caller job
forwards them, and a caller job cannot read environment-scoped secrets.
GitHub Environments, on the other hand, resolve inside the *called* jobs and
always against the **caller's** repo — protection rules, reviewers, and
environment URLs stay per-app.

`GITHUB_TOKEN` is never declared as a workflow_call secret; called jobs use
`github.token`, whose permissions are capped by the caller job's
`permissions:` block (see the ceilings below).

## How the workflows find their composite actions

The reusable workflows here reference this library's composite actions
**cross-repo by full path**:

```yaml
- uses: actions/checkout@<sha>            # caller repo
- uses: centrifuge/github-actions-lib/actions/build-app@main
  with:
    app-name: my-app
    node-version: '24'
```

Do **not** try to `actions/checkout` this library into a path and then
`uses: ./that-path/actions/x`. A `uses: ./…` local reference in a reusable
workflow is resolved against the reusable workflow's **own repo tree at its
pinned ref**, at *compile* time — not against the runtime workspace. The path
you checked out at runtime doesn't exist yet when GitHub resolves it, so the
called workflow fails to compile and every caller gets a `startup_failure`
with zero jobs. (The `job` context, likewise, only exposes
`status`/`container`/`services` — there is no `job.workflow_repository` /
`job.workflow_sha`, and `github.job_workflow_sha` cannot be used in a `uses:`
value anyway, since `uses:` takes no expressions.)

Because the composite `uses:` ref is a literal, the workflows and their
composites are versioned together by that ref. Consumers pin the *workflow*
`@main`; the workflows pin their composites `@main` too, so a library update
moves both. The one place a runtime checkout of this repo is legitimate is the
Lighthouse job, which reads the shared `lighthouserc.json` as a **file** (not
an action) — a normal `actions/checkout` into `.lib`, no `uses: ./`.

## Usage

Thin caller examples (see `apps-invest` / `apps-management` for the real ones):

```yaml
# .github/workflows/ci-checks.yml
name: '🔍 CI Checks'
on:
  pull_request:
jobs:
  ci:
    permissions:
      contents: read
      actions: read
    uses: centrifuge/github-actions-lib/.github/workflows/app-ci-checks.yml@main
    with:
      node-version: '24'
      codespell-skip: 'src/assets/**,.svg,.png'
```

```yaml
# .github/workflows/deploy-prod.yml
name: 🚀 Staging/Production Deploy
on:
  release:
    types: [released, prereleased]
permissions: {}
concurrency: build-${{ github.ref_name }}
jobs:
  build-deploy:
    permissions:
      contents: write        # ceiling: gh release upload in the build job
      deployments: write
    uses: centrifuge/github-actions-lib/.github/workflows/app-build-deploy-release.yml@main
    with:
      app-name: my-app
      node-version: '24'
      cloudflare-account-id: ${{ vars.CLOUDFLARE_ACCOUNT_ID }}
    secrets:
      cloudflare-api-token: ${{ secrets.CLOUDFLARE_API_TOKEN }}
```

### Caller permission ceilings

The caller job's `permissions:` is the ceiling for every job in the called
workflow (called jobs can only downgrade):

| Workflow | Caller job permissions |
|---|---|
| `app-ci-checks.yml` | `contents: read`, `actions: read` |
| `app-build-deploy-dev.yml` | `contents: read`, `deployments: write`, `id-token: write` |
| `app-build-deploy-release.yml` | `contents: write`, `deployments: write` |
| `app-rollback.yml` | `contents: read`, `deployments: write` |

### Contracts

Every input/secret/output is documented inline in each workflow's
`on.workflow_call` block — that block *is* the contract. Highlights:

- **`app-ci-checks.yml`** — required: `node-version`, `codespell-skip`.
  Overridable commands: `format-check-command`, `lint-command`,
  `audit-command`. `codespell-blocking: false` makes codespell advisory.
  `run-pinned-check: false` skips the pinact job. No secrets.
- **`app-build-deploy-dev.yml`** — required: `app-name`, `node-version`,
  `cloudflare-account-id`; secret `cloudflare-api-token`. `build-env` takes
  multiline `KEY=VALUE` pairs for the build step. `run-lighthouse: true`
  enables the LHCI job on PR previews, using this library's shared
  `lighthouserc.json` by default; set `lighthouse-config` to a caller-repo path
  to override. Output: `deployment-url`.
- **`app-build-deploy-release.yml`** — same core contract plus
  `bundle-name-prefix` (release zip name, defaults to `app-name`),
  `pool-cache-base-url`, `deploy-public-demo`, `production-url`. Outputs:
  `staging-url`, `production-url`. **A tag must be `prereleased` before it
  can be `released`** — production promotes the version staging uploaded.
- **`app-rollback.yml`** — required: `tag`, `app-name`,
  `cloudflare-account-id`; secret `cloudflare-api-token`.
  `environment: prod` (default) shifts traffic to the version tagged with
  `tag`; `environment: staging` re-uploads the release bundle behind the
  staging preview alias.

## Rules for consumers

- **Concurrency groups live in the caller** (they need caller context like
  the PR number). Called workflows define none.
- **Artifacts are scoped to the workflow run.** `build-app` uploads
  `<app-name>-build-<sha>` and v4 artifact names are immutable per run —
  never trigger two builds for the same `app-name` in one run.
- **Don't nest reusable workflows here.** A reusable workflow that itself
  calls another reusable workflow (e.g. the OSV scanner) fails the whole graph
  at compile time (`startup_failure`). That's why the OSV scan lives as a
  top-level job in each app's `ci-checks.yml` caller, not inside
  `app-ci-checks.yml`. Compose via composite *actions* (as the build/deploy
  workflows do), not nested `workflow_call`s.
- **Pinning**: consumers reference `@main`. If your repo runs pinact, add an
  ignore for this library (see the apps' `.pinact.yaml`); everything *inside*
  this library is SHA-pinned and `lib-ci.yml` enforces that.

## Making changes

Consumers track `@main`, so a merge here rolls out to every app's next
workflow run immediately:

1. Branch, edit, and open a PR here; `lib-ci.yml` (actionlint + pinact +
   yamllint) must pass.
2. For risky changes, point one app's caller at your branch
   (`...@your-branch`) in a draft PR to exercise the real pipeline, then
   revert the ref to `@main` before merging.
3. Behavior changes to deploy commands, artifact naming, or contracts must be
   coordinated with both app repos — grep their `.github/workflows/` callers
   first.
4. Tag notable states (`v1.0.0`, …) as human-readable restore points.
