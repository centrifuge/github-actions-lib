# Centrifuge GitHub Actions Library

Centralized CI/CD pipelines for Centrifuge app repos (`apps-invest`,
`apps-management`), published as **reusable workflows** (`workflow_call`).
Each app keeps only thin caller workflows that pass app-specific inputs and
its own secrets — pipeline changes happen once, here, and roll out to every
consumer.

## Contents

```
.github/workflows/
  app-ci-checks.yml             PR quality gate: format-n-lint, pnpm-audit (own check), TruffleHog, pinact
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

Do **not** `actions/checkout` this library into a path and reference
`uses: ./that-path/actions/x`: a `./` reference in a reusable workflow
resolves at compile time against the workflow's own repo tree, not the
runtime workspace, and every caller gets a `startup_failure` with zero jobs.

Workflows and composites both pin `@main`, so a library update moves them
together. The only legitimate runtime checkout of this repo is the Lighthouse
job reading `lighthouserc.json` as a plain file.

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

- **`app-ci-checks.yml`** — required: `node-version` only. Jobs:
  `format-n-lint` (defaults: `pnpm run format:check`, `pnpm run lint`,
  codespell over `src` with shared skip globs; `codespell-blocking: false`
  makes codespell advisory), `pnpm-audit` (own status check; default command
  routes through pnpm@11 while the apps are on pnpm <11), `secrets-scan`
  (TruffleHog), `pinned-actions-check` (pinact). Every command and glob is an
  input, so apps override only where they diverge. No secrets.
- **`app-build-deploy-dev.yml`** — required: `app-name`, `node-version`,
  `cloudflare-account-id`; secret `cloudflare-api-token`. Defaults:
  `build-args: --mode testnet`, Lighthouse on PR previews enabled
  (`run-lighthouse: true`) with this library's `lighthouserc.json`;
  `build-env` takes multiline `KEY=VALUE` pairs for the build step.
  Output: `deployment-url`.
- **`app-build-deploy-release.yml`** — same core contract plus
  `bundle-name-prefix` (release zip name, defaults to `app-name`),
  `pool-cache-base-url`, `deploy-public-demo`, `production-url`. Outputs:
  `staging-url`, `production-url`. **A tag must be `prereleased` before it
  can be `released`** — production promotes the version staging uploaded.
  Release bundles are immutable: rebuilding a tag whose bundle already exists
  fails; cut a new prerelease, or delete the asset from the release page to
  rebuild the same tag.
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

### Pipeline-only tool versions (not caller-overridable, by design)

Two version knobs are internal to this library's job execution, not to the
apps' own toolchain — they're deliberately hardcoded literals, not
`workflow_call` inputs, so apps can't drift them. (GitHub Actions can't source
an input `default:` from an external file at runtime, so this table is the
single place to look before changing any of them — not a machine-read config.)

| Where | Value | Why |
|---|---|---|
| `app-ci-checks.yml` → `pnpm-audit` job, `setup-app` `node-version` | `'22'` | The audit workaround below needs Node ≥22.13 (`node:sqlite`), independent of the app's own `node-version`. |
| `app-ci-checks.yml` → `audit-command` default, embedded `pnpm@11.0.0-rc.1` | RC, not stable | **Tested and confirmed (see github-actions-lib#6):** stable `pnpm@11.0.0` and current `latest` (`11.13.0`) both regressed to the retired classic audit endpoint (HTTP 410) — only this RC has the working bulk-advisory call. Do not bump this to "stable" or "latest" without re-testing against a real lockfile first. |
| `app-rollback.yml` → wrangler-install step, `setup-node` `node-version` | `'20'` | Only runs `npm install -g wrangler` + the wrangler CLI; unrelated to the app's toolchain. |

This is distinct from **`actions/setup-app`'s pnpm version**, which has no
hardcoded default at all — it's resolved from each app's own `package.json`
`"packageManager"` field, because that pnpm *is* the one developers use
locally (`pnpm install`, `pnpm build`, …) and should stay under app control.

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
