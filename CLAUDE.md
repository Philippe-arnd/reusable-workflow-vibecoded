# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Overview

This is a **GitHub Actions reusable workflow repository** for projects built with the Vibecoding method (React, Node, Express, Docker). There is no local build process — all workflows are consumed by other repositories via `workflow_call` with `secrets: inherit`.

Consumer repositories reference workflows as:
```yaml
jobs:
  validate:
    uses: Philippe-arnd/reusable-workflow-vibecoded/.github/workflows/reusable-pr-validation.yml@main
    secrets: inherit
    with:
      node-version: '24'
      ...
```

## Repository Structure

```
.github/workflows/
  reusable-pr-validation.yml       # Lint/build + Vitest + RLS tests + PR comment
  reusable-security-performance.yml # Gitleaks + Semgrep + bundle size + PR comment
  reusable-docker-validation.yml    # Docker build + health check (single or compose)
  reusable-dependency-review.yml    # CVE + license checks via dependency-review-action
  reusable-auto-merge.yml           # Squash-merge when all required checks pass

examples/
  kanban/   # Ready-to-use caller workflows for KanbanVibecoded
  journal/  # Ready-to-use caller workflows for JournalVibecoded
```

All reusable workflows are in `.github/workflows/`. The `examples/` directory contains production-ready caller files for each project — copy them to the target project's `.github/workflows/` to migrate from inline workflows to reusable ones.

## Workflow Architecture

### reusable-pr-validation.yml

Three parallel jobs feeding one report:
```
quick-checks  ──┐
vitest-tests  ──┤──► report (PR comment, sentinel: <!-- ci-test-results -->)
rls-tests     ──┘
```

Key inputs:
- `install-command` — override `npm ci` for monorepos (e.g. `npm ci && npm ci --prefix client`)
- `build-env` — multiline `KEY=VALUE` env vars for the build step (covers `VITE_ENCRYPTION_KEY`)
- `test-env` — multiline `KEY=VALUE` env vars for the test step (covers `BETTER_AUTH_SECRET` etc.)
- `db-migration-command` / `db-rls-setup-command` — optional; skipped when empty
- `rls-test-command` — when set, a separate parallel `rls-tests` job runs; leave empty to run RLS inline in `test-command`
- `coverage-dir` — path containing `coverage-summary.json` (default: `coverage`, use `server/coverage` for monorepos)
- `coverage-threshold` — minimum line coverage % (0 = no enforcement)

The DB setup sequence (always runs when `run-vitest: true`):
1. Create `app_user` (non-superuser so RLS policies are enforced)
2. Run `db-migration-command` if provided
3. Grant table/sequence permissions to `app_user` (includes `ALTER DEFAULT PRIVILEGES`)
4. Run `db-rls-setup-command` if provided

### reusable-security-performance.yml

```
secret-detection ──┐
security-scan    ──┤──► report (PR comment, sentinel: <!-- ci-security-report -->)
bundle-analysis  ──┘
```

Bundle analysis has two modes selected by whether baseline inputs are provided:
- **Absolute mode** (default): fail/warn when size exceeds `bundle-js-limit-kb` / `bundle-css-limit-kb`
- **Relative diff mode**: when `bundle-js-baseline-kb` > 0, shows delta + gzip sizes; warns if increase > `bundle-warn-threshold-pct`%

`bundle-client-dir` sets the working directory for the build and the `dist/assets/` path.

Semgrep base config is always `p/security-audit p/javascript p/react p/nodejs`. Add project-specific rules via `semgrep-extra-config` (e.g. `p/typescript .semgrep.yml`).

### reusable-docker-validation.yml

Single job with two mode branches via `use-compose: boolean`:
- **Single mode** (`use-compose: false`): builds with `docker build`, starts postgres + app containers, dynamic port detection via `docker port`, polls `health-endpoint`
- **Compose mode** (`use-compose: true`): runs `docker compose build` + `docker compose up -d`, health-checks by exec-ing into `compose-health-service`

### reusable-dependency-review.yml

Thin wrapper around `actions/dependency-review-action@v4`. Notable inputs:
- `allow-dependencies-licenses` — for MPL-2.0 transitive deps like lightningcss (see Kanban example)
- `warn-only` — set true to report without blocking

### reusable-auto-merge.yml

Triggered by `workflow_run` in the calling workflow. Verifies all `required-checks` (newline-separated names) are `completed/success` before squash-merging. Skips PRs that are draft, have conflicts, or carry `block-label`.

## Project Differences Encoded in Examples

| Concern | Kanban (`examples/kanban/`) | Journal (`examples/journal/`) |
|---|---|---|
| Install | `npm ci` | `npm ci` × 3 (root + client + server prefix) |
| Build env | `VITE_ENCRYPTION_KEY=ci-dummy-key` | none |
| DB admin password | `postgres` | `test_password` |
| Migrations | `npx drizzle-kit push --config=server/drizzle.config.js` | `npm run db:push --prefix server` |
| RLS setup | `node server/db/apply-rls.js` | `npm run db:rls --prefix server` |
| RLS tests | Separate job (`rls-test-command`) | Inline before vitest in `test-command` |
| Coverage path | `coverage/` | `server/coverage/` |
| Bundle mode | Absolute limits (JS 600KB, CSS 80KB) | Relative diff (JS 543KB baseline) |
| Bundle dir | `.` | `client` |
| Docker | Single Dockerfile | docker-compose |
| Semgrep extras | `.semgrep.yml` | `p/typescript .semgrep.yml` |

## Test Strategy: Parallel Validation

To validate a reusable workflow before replacing an inline one in a project:

1. **Copy** the relevant file from `examples/<project>/` to the project's `.github/workflows/` with a temporary name like `reusable-pr-validation.yml` alongside the existing `pr-validation.yml`
2. **Both workflows run in parallel** on each PR — they produce separate check results and PR comments
3. **Compare** the comments and check outcomes between inline and reusable
4. Once confident, **replace** the inline workflow by renaming/deleting the old file

The concurrency groups in the examples use different group names than the inline workflows, so they don't cancel each other during the test period.

## Action Versions

All reusable workflows use:
- `actions/checkout@v6`
- `actions/setup-node@v6`
- `actions/upload-artifact@v6`
- `actions/dependency-review-action@v4`
- `docker/setup-buildx-action@v3`
- `gitleaks/gitleaks-action@v2`
- `semgrep/semgrep-action@v1`
- `github/codeql-action/upload-sarif@v4`
- `peter-evans/find-comment@v4`
- `peter-evans/create-or-update-comment@v5`
- `actions/github-script@v8`
