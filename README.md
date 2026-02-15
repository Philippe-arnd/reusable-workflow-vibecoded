# Vibecoding Actions 🐙

Shared GitHub Actions and reusable workflows for projects built with the Vibecoding method (React, Node, Express, Docker). Consumed by KanbanVibecoded and JournalVibecoded, designed to accommodate any future project with the same stack.

## Reusable Workflows

### PR Validation

Lint/build + Vitest with coverage + optional parallel RLS tests + PR comment.

```yaml
jobs:
  validate:
    if: github.event.pull_request.draft == false
    uses: Philippe-arnd/vibecoding-actions/.github/workflows/reusable-pr-validation.yml@main
    secrets: inherit
    with:
      node-version: '24'
      postgres-db: my_test_db
      build-env: 'VITE_ENCRYPTION_KEY=ci-dummy-key'
      db-migration-command: npx drizzle-kit push --config=server/drizzle.config.js --force
      db-rls-setup-command: node server/db/apply-rls.js
      rls-test-command: node --test server/tests/test-rls.js
      test-env: |
        BETTER_AUTH_SECRET=ci-secret-minimum-32-chars
        BETTER_AUTH_URL=http://localhost:3000
      coverage-threshold: 80
```

Key inputs: `install-command` (monorepo override), `build-env` / `test-env` (multiline `KEY=VALUE`), `coverage-dir`, `rls-test-command` (empty = skip separate RLS job).

### Security & Performance

Gitleaks secret detection + Semgrep SAST + bundle size analysis + PR comment.

```yaml
jobs:
  security:
    if: github.event.pull_request.draft == false
    uses: Philippe-arnd/vibecoding-actions/.github/workflows/reusable-security-performance.yml@main
    secrets: inherit
    with:
      semgrep-extra-config: '.semgrep.yml'
      bundle-client-dir: '.'
      bundle-js-limit-kb: 600    # absolute mode
      # bundle-js-baseline-kb: 543  # relative diff mode (alternative)
```

### Docker Validation

Docker build + health check. Supports both single Dockerfile and docker-compose.

```yaml
jobs:
  docker:
    if: github.event.pull_request.draft == false
    uses: Philippe-arnd/vibecoding-actions/.github/workflows/reusable-docker-validation.yml@main
    secrets: inherit
    with:
      use-compose: false          # true for docker-compose projects
      image-tag: my-app:ci-test
      build-args: '--build-arg VITE_ENCRYPTION_KEY=ci-dummy-key'
      health-endpoint: /api/health
```

### Dependency Review

CVE and license checks via `actions/dependency-review-action`.

```yaml
jobs:
  review:
    uses: Philippe-arnd/vibecoding-actions/.github/workflows/reusable-dependency-review.yml@main
    secrets: inherit
    with:
      fail-on-severity: high
      allow-dependencies-licenses: 'pkg:npm/lightningcss'  # MPL-2.0 exemptions
```

### Auto Merge

Squash-merges a PR when all required checks pass. Triggered by `workflow_run`.

```yaml
on:
  workflow_run:
    workflows: ['PR Validation', 'Security & Performance', 'Dependency Review', 'Docker Validation']
    types: [completed]

jobs:
  auto-merge:
    uses: Philippe-arnd/vibecoding-actions/.github/workflows/reusable-auto-merge.yml@main
    secrets: inherit
    with:
      required-checks: |
        ✅ Quick Checks
        🧪 Vitest Tests
        🔒 RLS Tests
        🔑 Secret Detection
        🛡️ Security Scan
        🔎 Review Dependencies for Vulnerabilities
```

## Project Examples

Ready-to-use caller workflows for each project are in `examples/`:

```
examples/
  kanban/    # Single Dockerfile, absolute bundle limits, separate RLS job
  journal/   # Monorepo (client/server), docker-compose, relative bundle diff
```

Copy the relevant files to the target project's `.github/workflows/` to adopt the reusable workflows.

## Test Strategy

To validate before replacing existing inline workflows:

1. Copy an example caller into the project under a different filename (e.g. `reusable-pr-validation.yml`) alongside the existing `pr-validation.yml`
2. Both run in parallel on each PR, producing separate check entries and PR comments
3. Compare results, then replace the inline workflow once satisfied

The example callers use distinct concurrency group names so they don't cancel the inline workflows during testing.
