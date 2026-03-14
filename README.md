# Reusable Workflow Vibecoded 🚀

Reusable GitHub Actions workflows for projects built with the **Vibecoding** stack — React · Node · Express · Docker. Consumed today by **KanbanVibecoded** and **JournalVibecoded**, designed to accommodate any future project with the same stack.

> Consumer repos reference workflows as:
> ```yaml
> uses: Philippe-arnd/reusable-workflow-vibecoded/.github/workflows/<name>.yml@main
> secrets: inherit
> ```

---

## 📦 Workflow Overview

| Workflow | Trigger | Jobs |
|:---------|:--------|:-----|
| [🧪 PR Validation](#-pr-validation) | `pull_request` | Lint · Build · Vitest · RLS · Coverage |
| [🔐 Security & Performance](#-security--performance) | `pull_request` | Gitleaks · Semgrep · Bundle size |
| [🐳 Docker Validation](#-docker-validation) | `pull_request` | Build · Trivy scan · Health check |
| [🔎 Dependency Review](#-dependency-review) | `pull_request` | CVE check · License check |
| [🤖 Auto Merge](#-auto-merge) | `workflow_run` | Verify checks · Squash merge |
| [🧹 Branch Cleanup](#-branch-cleanup) | `pull_request [closed]` | Delete merged branch |
| [🏷️ Auto Label](#️-auto-label) | `pull_request` | Label by changed files |
| [📏 Update Bundle Baseline](#-update-bundle-baseline) | `push [main]` | Build · Measure · Commit baseline |

---

## 🧪 PR Validation

Three parallel jobs feeding a single PR comment updated on every push.

```
quick-checks  ──┐
vitest-tests  ──┤──► report (PR comment  ·  sentinel: <!-- ci-test-results -->)
rls-tests     ──┘
```

**Key inputs**

| Input | Default | Description |
|:------|:--------|:------------|
| `install-command` | `npm ci` | Override for monorepos |
| `build-env` | — | Multiline `KEY=VALUE` injected into the build step |
| `test-env` | — | Multiline `KEY=VALUE` injected into the test step |
| `db-migration-command` | — | e.g. `npx drizzle-kit push` — skipped when empty |
| `db-rls-setup-command` | — | e.g. `node server/db/apply-rls.js` — skipped when empty |
| `rls-test-command` | — | Empty = RLS inline · Set = separate parallel job |
| `coverage-dir` | `coverage` | Use `server/coverage` for monorepos |
| `coverage-threshold` | `0` | Minimum line coverage % — `0` = no enforcement |

**DB setup sequence** (runs in both `vitest-tests` and `rls-tests` via the `setup-db` composite action):

| Step | Description |
|:-----|:------------|
| 1 | Create `app_user` (non-superuser — RLS policies are enforced) |
| 2 | Run `db-migration-command` if provided |
| 3 | Grant table/sequence permissions to `app_user` + `ALTER DEFAULT PRIVILEGES` |
| 4 | Run `db-rls-setup-command` if provided |

```yaml
jobs:
  validate:
    if: github.event.pull_request.draft == false
    uses: Philippe-arnd/reusable-workflow-vibecoded/.github/workflows/reusable-pr-validation.yml@main
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

---

## 🔐 Security & Performance

Three parallel jobs, results uploaded to the GitHub **Security** tab as SARIF.

```
secret-detection ──┐
security-scan    ──┤──► report (PR comment  ·  sentinel: <!-- ci-security-report -->)
bundle-analysis  ──┘
```

**Key inputs**

| Input | Default | Description |
|:------|:--------|:------------|
| `gitleaks-config-path` | `.gitleaks.toml` | Empty = built-in Gitleaks rules |
| `semgrep-extra-config` | — | Extra rulesets appended to `p/security-audit p/javascript p/react p/nodejs` |
| `bundle-client-dir` | `.` | Directory containing `dist/assets/` after build |
| `bundle-js-limit-kb` | `600` | Absolute mode — fail/warn when exceeded |
| `bundle-js-baseline-kb` | `0` | Set > 0 to activate **relative diff mode** |
| `bundle-warn-threshold-pct` | `10` | Relative diff mode — warn if increase > X% |

**Bundle modes**

| Mode | Activation | Behaviour |
|:-----|:-----------|:----------|
| Absolute | `bundle-js-baseline-kb: 0` (default) | ⚠️ when size exceeds limit |
| Relative diff | `bundle-js-baseline-kb > 0` | Shows Δ KB and % vs baseline · ⚠️ if increase > threshold |

```yaml
jobs:
  security:
    if: github.event.pull_request.draft == false
    uses: Philippe-arnd/reusable-workflow-vibecoded/.github/workflows/reusable-security-performance.yml@main
    secrets: inherit
    with:
      semgrep-extra-config: '.semgrep.yml'
      bundle-client-dir: '.'
      bundle-js-limit-kb: 600        # absolute mode
      # bundle-js-baseline-kb: 543   # relative diff mode (alternative)
```

---

## 🐳 Docker Validation

Supports both single Dockerfile and Docker Compose. Includes Trivy CVE image scanning with SARIF upload.

**Key inputs**

| Input | Default | Description |
|:------|:--------|:------------|
| `use-compose` | `false` | `true` for docker-compose projects |
| `image-tag` | `app:ci-test` | Tag for the built image (single mode) |
| `health-endpoint` | `/api/health` | HTTP path polled to confirm the app is up |
| `run-trivy-scan` | `true` | Scan the built image for OS/dependency CVEs |
| `trivy-severity` | `CRITICAL,HIGH` | Severity levels to report |
| `trivy-fail-on-finding` | `false` | Fail the job on CVE findings (warn-only by default) |
| `trivy-compose-image` | — | Image name to scan in compose mode — empty = skip Trivy |

```yaml
jobs:
  docker:
    if: github.event.pull_request.draft == false
    uses: Philippe-arnd/reusable-workflow-vibecoded/.github/workflows/reusable-docker-validation.yml@main
    secrets: inherit
    with:
      use-compose: false
      image-tag: my-app:ci-test
      build-args: '--build-arg VITE_ENCRYPTION_KEY=ci-dummy-key'
      health-endpoint: /api/health
      # trivy-compose-image: myapp-server:latest  # compose mode
```

---

## 🔎 Dependency Review

Thin wrapper around `actions/dependency-review-action@v4`.

**Key inputs**

| Input | Default | Description |
|:------|:--------|:------------|
| `fail-on-severity` | `high` | `low` / `moderate` / `high` / `critical` |
| `allow-licenses` | `MIT Apache-2.0 BSD…` | Allowed SPDX identifiers |
| `allow-dependencies-licenses` | — | `pkg:npm/<pkg>` exemptions for non-standard licenses (e.g. MPL-2.0 transitive deps like lightningcss) |
| `warn-only` | `false` | Report without blocking the PR |

```yaml
jobs:
  review:
    uses: Philippe-arnd/reusable-workflow-vibecoded/.github/workflows/reusable-dependency-review.yml@main
    secrets: inherit
    with:
      allow-dependencies-licenses: 'pkg:npm/lightningcss'
```

---

## 🤖 Auto Merge

Triggered by `workflow_run`. Verifies all `required-checks` are `completed/success` then squash-merges. Skips drafts, PRs with conflicts, and PRs carrying `block-label`.

**Key inputs**

| Input | Default | Description |
|:------|:--------|:------------|
| `required-checks` | — | Newline-separated check names that must all pass |
| `conditional-checks` | — | Block only if they ran — skip if the check run is absent |
| `merge-method` | `squash` | `squash` / `merge` / `rebase` |
| `block-label` | `major-update` | PRs with this label require manual review |

```yaml
on:
  workflow_run:
    workflows: ['PR Validation', 'Security & Performance', 'Dependency Review']
    types: [completed]
    branches: [main]

jobs:
  auto-merge:
    uses: Philippe-arnd/reusable-workflow-vibecoded/.github/workflows/reusable-auto-merge.yml@main
    secrets: inherit
    with:
      required-checks: |
        ✅ Quick Checks
        🧪 Vitest Tests
        🔑 Secret Detection
        🛡️ Security Scan
        🔎 Review Dependencies for Vulnerabilities
      conditional-checks: |
        🐳 Build & Validate Docker Image
```

---

## 🧹 Branch Cleanup

Deletes the source branch automatically after a PR is merged. Protects reserved branch names (`main`, `master`, `develop`, `staging`, and any extra names via `protected-branches`).

```yaml
on:
  pull_request:
    branches: [main]
    types: [closed]

jobs:
  cleanup:
    uses: Philippe-arnd/reusable-workflow-vibecoded/.github/workflows/reusable-branch-cleanup.yml@main
    secrets: inherit
    with:
      base-branch: main
```

---

## 🏷️ Auto Label

Labels PRs automatically based on changed files using `actions/labeler@v5`. Requires a `.github/labeler.yml` in the calling repo.

**Key inputs**

| Input | Default | Description |
|:------|:--------|:------------|
| `configuration-path` | `.github/labeler.yml` | Path to the labeler config file |
| `sync-labels` | `true` | Remove labels when files no longer match |

```yaml
on:
  pull_request:
    types: [opened, synchronize, reopened]

jobs:
  label:
    uses: Philippe-arnd/reusable-workflow-vibecoded/.github/workflows/reusable-auto-label.yml@main
    secrets: inherit
```

> Example `labeler.yml` configurations for Kanban and Journal are provided in `examples/`.

---

## 📏 Update Bundle Baseline

Triggered on `push` to `main` after a PR is merged. Rebuilds the bundle, measures JS/CSS sizes, and commits updated `bundle-js-baseline-kb` / `bundle-css-baseline-kb` values back to the specified workflow file. Only relevant for consumers using **relative diff mode**.

> ℹ️ No infinite loop risk — commits made by `GITHUB_TOKEN` do not re-trigger workflows.

**Key inputs**

| Input | Default | Description |
|:------|:--------|:------------|
| `bundle-client-dir` | `.` | Directory containing `dist/assets/` after build |
| `workflow-file` | — | Path to the workflow file to update (e.g. `.github/workflows/security-performance.yml`) |

```yaml
on:
  push:
    branches: [main]
    paths:
      - 'client/src/**'
      - 'client/package-lock.json'

jobs:
  update-baseline:
    uses: Philippe-arnd/reusable-workflow-vibecoded/.github/workflows/reusable-update-bundle-baseline.yml@main
    secrets: inherit
    with:
      install-command: npm ci --prefer-offline
      build-command: npm run build
      bundle-client-dir: client
      workflow-file: .github/workflows/security-performance.yml
```

---

## ⚙️ Composite Actions

### `setup-db`

Used internally by `reusable-pr-validation.yml` in both the `vitest-tests` and `rls-tests` jobs. Eliminates duplication across the two jobs.

| Step | Action |
|:-----|:-------|
| 1 | Install PostgreSQL client |
| 2 | Wait for the service to be ready |
| 3 | Create `app_user` + grant `CONNECT` |
| 4 | Run `db-migration-command` (if set) |
| 5 | Grant table/sequence permissions |
| 6 | Run `db-rls-setup-command` (if set) |

---

## 📁 Project Examples

Ready-to-use caller files in `examples/` — copy to the target project's `.github/workflows/`.

| File | Kanban | Journal |
|:-----|:------:|:-------:|
| `pr-validation.yml` | ✅ | ✅ |
| `security-performance.yml` | ✅ | ✅ |
| `docker-validation.yml` | ✅ | ✅ |
| `dependency-review.yml` | ✅ | ✅ |
| `auto-merge.yml` | ✅ | ✅ |
| `branch-cleanup.yml` | ✅ | ✅ |
| `auto-label.yml` + `labeler.yml` | ✅ | ✅ |
| `update-bundle-baseline.yml` | — | ✅ |

**Key differences between projects**

| Concern | Kanban | Journal |
|:--------|:-------|:--------|
| Install | `npm ci` | `npm ci` × 3 (root + client + server) |
| Build env | `VITE_ENCRYPTION_KEY=ci-dummy-key` | — |
| DB admin password | `postgres` | `test_password` |
| Migrations | `npx drizzle-kit push` | `npm run db:push --prefix server` |
| RLS setup | `node server/db/apply-rls.js` | `npm run db:rls --prefix server` |
| RLS tests | Separate parallel job | Inline in `test-command` |
| Coverage path | `coverage/` | `server/coverage/` |
| Bundle mode | Absolute (JS 600 KB · CSS 80 KB) | Relative diff (JS 543 KB baseline) |
| Bundle dir | `.` | `client/` |
| Docker | Single Dockerfile | docker-compose |
| Semgrep extras | `.semgrep.yml` | `p/typescript .semgrep.yml` |
| Bundle baseline auto-update | — | ✅ |

---

## 🧪 Test Strategy

To validate a reusable workflow before replacing an existing inline one:

1. **Copy** the example file to the project under a temporary name (e.g. `reusable-pr-validation.yml`) alongside the existing `pr-validation.yml`
2. **Both run in parallel** on each PR — separate check entries and separate PR comments
3. **Compare** outcomes, then replace the inline workflow once satisfied

The example callers use distinct concurrency group names so they don't cancel each other during the parallel test period.

---

## 🔖 Action Versions

| Action | Version |
|:-------|:--------|
| `actions/checkout` | `v6` |
| `actions/setup-node` | `v6` |
| `actions/upload-artifact` | `v6` |
| `actions/labeler` | `v5` |
| `actions/dependency-review-action` | `v4` |
| `actions/github-script` | `v8` |
| `docker/setup-buildx-action` | `v3` |
| `aquasecurity/trivy-action` | `0.28.0` |
| `gitleaks/gitleaks-action` | `v2` |
| `semgrep/semgrep-action` | `v1` |
| `github/codeql-action/upload-sarif` | `v4` |
| `peter-evans/find-comment` | `v4` |
| `peter-evans/create-or-update-comment` | `v5` |
