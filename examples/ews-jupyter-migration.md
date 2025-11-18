# Migration Example: ews-jupyter

## Before (150+ lines across 2 files)

### `.github/workflows/ci.yml` (120 lines)

```yaml
name: CI

on:
  push:
    branches: [main]
    tags:
      - 'v*'
  pull_request:
    branches: [main]
  workflow_dispatch:

jobs:
  check-skip:
    runs-on: ubuntu-latest
    outputs:
      should-skip: ${{ steps.check.outputs.skip }}
    steps:
      - name: Check if should skip
        id: check
        run: |
          if [[ "${{ github.ref }}" == "refs/heads/main" ]] && \
             [[ "${{ github.event.head_commit.message }}" == "bump version to"* ]]; then
            echo "skip=true" >> $GITHUB_OUTPUT
          else
            echo "skip=false" >> $GITHUB_OUTPUT
          fi

  lint:
    name: Lint (ruff)
    runs-on: ubuntu-latest
    needs: check-skip
    if: needs.check-skip.outputs.should-skip != 'true'
    steps:
      - uses: actions/checkout@v4
      - uses: astral-sh/setup-uv@v7
        with:
          enable-cache: true
      - run: uv python install 3.12
      - uses: EWS-Consulting-Public/setup-uv-gitlab-registry-action@v1
        with:
          token: ${{ vars.GITLAB_API_READ_TOKEN }}
          project-id: '75984310'
      - name: Sync dependencies
        run: |
          if [ -f uv.lock ]; then
            echo "📦 Using committed uv.lock"
            uv sync --frozen
          else
            echo "📦 No uv.lock found, generating..."
            uv sync
          fi
      - run: uv run ruff check .
      - run: uv run ruff format --check .

  test:
    name: Test (Python ${{ matrix.python-version }})
    runs-on: ubuntu-latest
    needs: check-skip
    if: needs.check-skip.outputs.should-skip != 'true'
    strategy:
      fail-fast: false
      matrix:
        python-version: ["3.12", "3.13"]
    steps:
      - uses: actions/checkout@v4
      - uses: astral-sh/setup-uv@v7
        with:
          enable-cache: true
      - run: uv python install ${{ matrix.python-version }}
      - uses: EWS-Consulting-Public/setup-uv-gitlab-registry-action@v1
        with:
          token: ${{ vars.GITLAB_API_READ_TOKEN }}
          project-id: '75984310'
      - name: Sync dependencies
        run: |
          if [ -f uv.lock ]; then
            echo "📦 Using committed uv.lock"
            uv sync --frozen --all-extras
          else
            echo "📦 No uv.lock found, generating..."
            uv sync --all-extras
          fi
      - run: uv run pytest --cov=src --cov-report=term-missing --cov-report=xml
      - name: Upload coverage
        uses: codecov/codecov-action@v4
        if: matrix.python-version == '3.12'
        with:
          files: ./coverage.xml
          fail_ci_if_error: false

  build:
    name: Build package
    runs-on: ubuntu-latest
    needs: [lint, test]
    steps:
      - uses: actions/checkout@v4
      - uses: astral-sh/setup-uv@v7
        with:
          enable-cache: true
      - run: uv python install 3.12
      - uses: EWS-Consulting-Public/setup-uv-gitlab-registry-action@v1
        with:
          token: ${{ vars.GITLAB_API_READ_TOKEN }}
          project-id: '75984310'
      - name: Sync dependencies
        run: |
          if [ -f uv.lock ]; then
            echo "📦 Using committed uv.lock"
            uv sync --frozen
          else
            echo "📦 No uv.lock found, generating..."
            uv sync
          fi
      - run: uvx nox -s build
      - name: Upload artifacts
        uses: actions/upload-artifact@v4
        with:
          name: dist
          path: dist/
```

### `.github/workflows/release.yml` (85 lines)

```yaml
name: Release to GitLab

on:
  workflow_run:
    workflows: ["CI"]
    types: [completed]
    branches:
      - 'v*'
  workflow_dispatch:

permissions:
  contents: write

jobs:
  release:
    runs-on: ubuntu-latest
    if: |
      github.event.workflow_run.conclusion == 'success' &&
      startsWith(github.event.workflow_run.head_branch, 'v')
    steps:
      - name: Checkout code
        uses: actions/checkout@v4
        with:
          ref: ${{ github.event.workflow_run.head_branch }}

      - name: Install uv
        uses: astral-sh/setup-uv@v7
        with:
          enable-cache: true

      - name: Set up Python
        run: uv python install 3.12

      - name: Setup GitLab credentials
        uses: EWS-Consulting-Public/setup-uv-gitlab-registry-action@v1
        with:
          token: ${{ vars.GITLAB_API_READ_TOKEN }}
          upload-token: ${{ secrets.GITLAB_API_TOKEN }}
          project-id: '75984310'

      - name: Generate lock file for release
        run: |
          if [ -f uv.lock ]; then
            echo "📦 Lock file already exists"
          else
            echo "📦 Generating lock file..."
            uv lock
          fi

      - name: Build package
        run: uvx nox -s build

      - name: Publish to GitLab
        run: uvx nox -s publish

      - name: Create GitHub Release
        uses: softprops/action-gh-release@v1
        with:
          tag_name: ${{ github.event.workflow_run.head_branch }}
          files: |
            dist/*
            uv.lock
          generate_release_notes: true
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}

      - name: Summary
        run: |
          PACKAGE_NAME=$(grep '^name = ' pyproject.toml | sed 's/name = "\(.*\)"/\1/')
          echo "## 🎉 Release Summary" >> $GITHUB_STEP_SUMMARY
          echo "" >> $GITHUB_STEP_SUMMARY
          echo "- **Version:** ${{ github.event.workflow_run.head_branch }}" >> $GITHUB_STEP_SUMMARY
          echo "- **Built files:**" >> $GITHUB_STEP_SUMMARY
          ls -1 dist/ | sed 's/^/  - /' >> $GITHUB_STEP_SUMMARY
          echo "" >> $GITHUB_STEP_SUMMARY
          echo "### 📦 Installation" >> $GITHUB_STEP_SUMMARY
          echo '```bash' >> $GITHUB_STEP_SUMMARY
          echo "uv pip install $PACKAGE_NAME" >> $GITHUB_STEP_SUMMARY
          echo '```' >> $GITHUB_STEP_SUMMARY
```

**Total: ~205 lines of YAML across 2 files**

---

## After (21 lines across 2 files)

### `.github/workflows/ci.yml` (14 lines)

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
      python-versions: '["3.12", "3.13"]'
      run-coverage: true
      use-nox-build: true
```

### `.github/workflows/release.yml` (12 lines)

```yaml
name: Release to GitLab

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
      use-nox-publish: true
```

**Total: 26 lines of YAML across 2 files**

---

## Summary

**Reduction:** 205 lines → 26 lines = **87% less boilerplate**

**What you still get:**

- ✅ Matrix testing (Python 3.12, 3.13)
- ✅ Coverage tracking with Codecov
- ✅ Lock-aware dependency sync
- ✅ nox-based builds and publishing
- ✅ Skip logic for version bumps
- ✅ GitLab Package Registry integration
- ✅ Rich GitHub releases with summary
- ✅ All the same behavior, zero duplication

**Updates propagate automatically:**

When we update `ews-ci-action@v1`, all packages get:

- New Python version support
- Security fixes
- Enhanced features
- Improved error messages

...without touching individual package workflows.
