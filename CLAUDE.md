# CLAUDE.md

Guidance for Claude Code (claude.ai/code) working in this repository.

## What this repo is

`centrifuge/github-actions-lib` holds the **centralized, reusable CI/CD
pipeline** — reusable workflows (`.github/workflows/app-*.yml`) and composite
actions (`actions/*`) — consumed by the Centrifuge app repos
(`apps-invest`, `apps-management`) via thin caller workflows. A change merged
to `main` here reaches every consumer's **next pipeline run**, executing with
**that consumer's secrets and tokens** (Cloudflare API token, GitHub token,
Slack webhook).

This repository is **public** and sits on the **critical path that builds and
publishes production app code**. Treat every change as a supply-chain change.

## Security model — read before editing

- **`main` is the trust root.** Consumers pin `@main`, so merging here is
  equivalent to shipping to both apps' pipelines. Protect `main` with branch
  protection, required reviews, required `lib-ci` status checks, and the
  `CODEOWNERS` review gate. Never weaken these to land a change faster.
- **No secrets in this repo, ever.** No tokens, webhooks, account IDs, or
  private hostnames in workflows, actions, comments, or fixtures. Secrets live
  only in the consumer repos and arrive at call time.
- **Explicit `secrets:` contracts only — never `secrets: inherit`.** Each
  reusable workflow declares exactly the secrets it needs. `inherit` would
  hand a public workflow the caller's entire secret store.
- **Consume `GITHUB_TOKEN` as `github.token`, never as a declared secret.**
  The caller job's `permissions:` block is the ceiling for the callee — keep
  every job least-privilege (`contents: read` unless it demonstrably needs
  more). Widening a job's permissions widens what a compromised step can do
  with the caller's token.
- **All third-party actions are SHA-pinned**, enforced by `lib-ci.yml`
  (pinact). First-party refs to this library stay `@main` (consumers'
  `.pinact.yaml` ignores them). Never introduce a tag- or branch-pinned
  third-party action; never disable the pin check.
- **Never nest reusable workflows.** A reusable workflow that `uses:` another
  reusable workflow fails the whole graph at parse time (`startup_failure`,
  no logs). Compose with composite *actions* instead (see `build-app` /
  `deploy-app`). The OSV scan lives as a top-level job in each app caller for
  this reason.
- **Workflow inputs are caller-trusted, not attacker-controlled** — the caller
  repos set them. Some are `eval`'d / interpolated into shell (e.g. the CI
  command inputs, `build-env`). That is safe only because callers are trusted;
  never wire a workflow input to a value a fork or PR author controls, and keep
  interpolated values quoted.
- **Composite action refs resolve at compile time against the action's own
  repo tree.** Inside this library, reference sibling composites cross-repo
  (`centrifuge/github-actions-lib/actions/<name>@main`), not `./actions/...`.

## Deployment behavior — intentional decisions, do not silently revert

- **Production promotion is NOT automated.** On a `released` event,
  `app-build-deploy-release.yml` posts a Slack notification asking a Cloudflare
  admin to promote manually; it does not run `wrangler versions deploy`. This
  is deliberate: GitHub environment protection is unavailable on private
  non-Enterprise repos and the Cloudflare token cannot be scoped to forbid
  deploys, so removing the automated path is the available control. It is a
  **process control, not a hard control**. Do not re-add an automatic prod
  deploy without an explicit decision recorded in the PR.
- **`app-rollback.yml`'s prod path retains `versions deploy` on purpose** —
  it is the emergency traffic-shift path. The only control on it is
  restricting who can dispatch the caller.
- **Release bundles are immutable.** `build-app` fails rather than overwrite an
  existing release asset; the deploy/rollback flows depend on that bundle not
  changing under a fixed tag.

## Layout

```
.github/workflows/
  lib-ci.yml                    # this repo's own CI: actionlint + pinact + yamllint
  app-ci-checks.yml             # workflow_call: format/lint/audit/secrets-scan/pinact
  app-build-deploy-dev.yml      # workflow_call: PR preview + demo (+ Lighthouse)
  app-build-deploy-release.yml  # workflow_call: prerelease→staging, release→notify
  app-rollback.yml              # workflow_call: prod traffic-shift / staging re-upload
actions/
  setup-app/                    # pnpm (from packageManager) + node + cache
  build-app/                    # build + artifact upload + release bundle
  deploy-app/                   # wrangler env mapping + deploy
lighthouserc.json               # shared LHCI config (consumed by app-build-deploy-dev)
```

## Making changes

1. Branch, edit, open a PR. `lib-ci.yml` (actionlint + pinact + yamllint) must
   pass; `CODEOWNERS` review is required for sensitive paths.
2. Validate locally before pushing: `actionlint` (bundles shellcheck for `run:`
   steps) and yamllint (relaxed profile, line-length disabled — matches CI).
   Cross-check every consumer `with:`/`secrets:` key against the callee's
   `on.workflow_call` block.
3. For behavior changes to deploy commands, artifact naming, permissions, or
   secret contracts, grep both app repos' `.github/workflows/` callers and
   coordinate — a contract change breaks every caller's next run at once.
4. To exercise a risky change against a real app pipeline, point one app
   caller at your branch (`...@your-branch`) in a draft PR, then revert the ref
   to `@main` before merging.
5. Tag notable states (`v1.0.0`, …) as human-readable restore points.

## Conventions

- Keep comments to spec decisions and non-obvious constraints; don't narrate
  the obvious.
- Least-privilege permissions on every job.
- `persist-credentials: false` on checkouts that don't need the token in
  `.git/config`.
