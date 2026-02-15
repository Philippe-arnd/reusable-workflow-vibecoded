# Vibecoding Actions 🐙

Shared GitHub Actions and Workflows for projects built with the Vibecoding method (React, Node, Express, Docker).

## Workflows

### Reusable PR Validation

A comprehensive validation workflow including:
- Linting and Building
- Testing with Vitest (and optional coverage)
- Database testing (PostgreSQL/RLS)
- PR Commenting with results

## Usage

```yaml
jobs:
  validate:
    uses: Philippe-arnd/vibecoding-actions/.github/workflows/reusable-pr-validation.yml@main
    with:
      node-version: 24
      run-tests: true
```
