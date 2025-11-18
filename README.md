# ews-ci-action

**Reusable CI/CD workflows and actions for EWS Python packages.**

Centralized GitHub Actions that eliminate duplication across EWS Python packages by providing:

- Standard CI workflow (lint, test with matrix, coverage, build)
- Standard release workflow (publish to GitLab Package Registry + GitHub Releases)
- Composite action for common setup steps

## Features

- ✅ **Matrix testing** across multiple Python versions (3.12, 3.13, 3.14)
- ✅ **Coverage tracking** with Codecov integration
- ✅ **Lock-aware sync** - respects committed `uv.lock` files
- ✅ **nox-based builds** - supports `nox -s build` and `nox -s publish`
- ✅ **GitLab Package Registry** - automatic authentication and publishing
- ✅ **Rich GitHub releases** - includes wheels, lockfile, and summary
- ✅ **CalVer-aware** release automation
- ✅ **Skip logic** for version bump commits (no duplicate runs)

## Quick Start

### Use Reusable CI Workflow

In your package's `.github/workflows/ci.yml`:

```yaml
name: CI

on:
  push:
    branches: [main]
    tags:
      - 'v*'
  pull_request:
    branches: [main]

jobs:
  ci:
    uses: EWS-Consulting-Public/ews-ci-action/.github/workflows/ci.yml@v1
    with:
      python-versions: '["3.12", "3.13"]'  # Matrix testing
      run-coverage: true  # Upload to Codecov
      use-nox-build: true  # Use nox for builds
```

### Use Reusable Release Workflow

In your package's `.github/workflows/release.yml`:

```yaml
name: Release

on:
  workflow_run:
    workflows: ["CI"]
    types: [completed]
    branches:
      - 'v*'

jobs:
  release:
    uses: EWS-Consulting-Public/ews-ci-action/.github/workflows/release.yml@v1
    with:
      use-nox-publish: true  # Use nox for publishing
```

That's it! Your package now has:

- Matrix testing across Python versions
- Coverage tracking (Codecov)
- Lock-aware dependency management
- Automated builds (nox or uv)
- Automated releases (GitLab + GitHub)

## Available Workflows

### `ci.yml` - Continuous Integration

**Inputs:**

- `python-versions` (default: `'["3.12"]'`) - JSON array of Python versions for matrix testing
- `lint-python-version` (default: `"3.12"`) - Python version for lint job
- `run-lint` (default: `true`) - Run ruff format + check
- `run-tests` (default: `true`) - Run pytest across all versions
- `run-coverage` (default: `true`) - Collect coverage and upload to Codecov
- `run-build` (default: `true`) - Build wheel package
- `use-nox-build` (default: `true`) - Use `uvx nox -s build` instead of `uv build`
- `upload-artifacts` (default: `true`) - Upload build artifacts

**Jobs:**

1. `check-skip` - Detects version bump commits and skips duplicate CI runs
2. `lint` - Runs `ruff check .` and `ruff format --check .`
3. `test` - Matrix job: runs `pytest` (with optional coverage) across Python versions
4. `build` - Runs `uvx nox -s build` or `uv build --wheel` and uploads artifacts

**What changed vs minimal wrapper:**
- ✅ Matrix testing (not single version)
- ✅ Coverage + Codecov upload
- ✅ Lock-aware sync (`uv.lock` respected)
- ✅ nox build support
- ✅ Build waits for lint+test (proper dependencies)

### `release.yml` - Automated Releases

**Inputs:**

- `python-version` (default: `"3.12"`) - Python version for build
- `use-nox-publish` (default: `true`) - Use `uvx nox -s publish` instead of direct twine

**Required secrets/variables:**
- `GITLAB_API_TOKEN` - For publishing (Maintainer role) - **repository secret**
- `GITLAB_API_READ_TOKEN` - For installing dependencies - **repository variable**
- `GITLAB_PACKAGE_REGISTRY_URL` - Registry URL - **repository variable**

**What it does:**

1. Checks out tagged ref
2. Sets up Python + uv + GitLab credentials
3. Ensures `uv.lock` exists (generates if missing)
4. Downloads build artifacts from CI
5. Publishes to GitLab via `nox -s publish` or `twine`
6. Creates GitHub release with wheels + lockfile
7. Writes rich summary with installation instructions

**What changed vs minimal wrapper:**
- ✅ Auto-generates lockfile if missing
- ✅ Uses `setup-uv-gitlab-registry-action` with upload token
- ✅ nox publish support
- ✅ Rich GitHub release summary
- ✅ Uses `softprops/action-gh-release` (not `gh` CLI)

## Composite Action

Use the `setup-ews-ci` composite action directly for custom workflows:

```yaml
steps:
  - uses: actions/checkout@v4
  
  - uses: EWS-Consulting-Public/ews-ci-action/.github/actions/setup-ews-ci@v1
    with:
      python-version: "3.12"
      gitlab-token: ${{ vars.GITLAB_API_READ_TOKEN }}
  
  - run: uv run pytest
```

**What it does:**
1. Installs `uv` with caching
2. Installs Python version
3. Configures GitLab Package Registry (via `setup-uv-gitlab-registry-action`)
4. Optionally runs lock-aware `uv sync` (checks for `uv.lock`, uses `--frozen` if present)

## Examples

See [`examples/`](examples/) for complete workflow templates:

- [`basic-package.yml`](examples/basic-package.yml) - Matrix testing with coverage
- [`matrix-testing.yml`](examples/matrix-testing.yml) - Test Python 3.12, 3.13, 3.14
- [`custom-steps.yml`](examples/custom-steps.yml) - Add custom validation steps
- [`release-basic.yml`](examples/release-basic.yml) - nox-based release

## Migration Guide

### From Duplicated Workflows

**Before** (in each package):

```yaml
# .github/workflows/ci.yml (150+ lines)
name: CI
on: [push, pull_request]
jobs:
  check-skip: ...
  lint: ...
  test:
    strategy:
      matrix:
        python-version: ["3.12", "3.13"]
    steps:
      - uses: astral-sh/setup-uv@v7
      - uses: EWS-Consulting-Public/setup-uv-gitlab-registry-action@v1
      - run: uv sync --frozen
      - run: uv run pytest --cov
      - uses: codecov/codecov-action@v4
  build:
    steps:
      - run: uvx nox -s build
```
```

**After**:

```yaml
# .github/workflows/ci.yml (8 lines!)
name: CI
on: [push, pull_request]
jobs:
  ci:
    uses: EWS-Consulting-Public/ews-ci-action/.github/workflows/ci.yml@v1
    secrets: inherit
```

## Development

**Testing changes locally:**

1. Push changes to a branch
2. Update a test package to use `@branch-name` instead of `@v1`
3. Trigger workflow and verify behavior

**Releasing a new version:**

```bash
# Tag the action
git tag -a v1.0.0 -m "Release v1.0.0"
git push --tags

# Update major version tag (so @v1 always points to latest v1.x.x)
git tag -fa v1 -m "Update v1 to v1.0.0"
git push --force --tags
```

## License

MIT

## Support

Questions? Issues? See [EWS Bootstrap Documentation](https://github.com/EWS-Consulting-Private/ews-bootstrap)

