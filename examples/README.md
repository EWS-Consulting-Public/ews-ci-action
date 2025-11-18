# Example Workflows

This directory contains example workflow configurations for different use cases.

## Available Examples

### 1. [`basic-package.yml`](basic-package.yml)

Minimal setup for a standard EWS Python package.

**Use when:**

- You have a simple package with standard structure
- You want the default Python 3.12
- You're using the main GitLab package registry

### 2. [`matrix-testing.yml`](matrix-testing.yml)

Test across multiple Python versions.

**Use when:**
- You support Python 3.12, 3.13, 3.14+
- You want to verify compatibility across versions
- You need to test on different OS platforms (Linux, macOS, Windows)

### 3. [`custom-steps.yml`](custom-steps.yml)

Add custom validation or build steps beyond standard CI.

**Use when:**
- You need additional checks (e.g., notebook validation, docs build)
- You want to run package-specific scripts
- You need custom artifact handling

### 4. [`release-only.yml`](release-only.yml)

Package with CI but manual release process.

**Use when:**
- You want automated testing but manual publishing
- You're using a different package registry
- You need approval gates before releases

## How to Use

1. Copy the example that matches your needs
2. Rename to `.github/workflows/ci.yml` (or `release.yml`) in your package
3. Adjust inputs to match your requirements
4. Commit and push

## Common Customizations

### Change Python Version

```yaml
with:
  python-version: "3.13"
```

### Skip Tests (lint only)

```yaml
with:
  run-tests: false
```

### Different GitLab Project

```yaml
with:
  gitlab-project-id: "12345678"
```

### Add Custom Steps After CI

```yaml
jobs:
  ci:
    uses: EWS-Consulting-Public/ews-ci-action/.github/workflows/ci.yml@v1
    secrets: inherit

  custom-validation:
    needs: ci
    runs-on: ubuntu-latest
    steps:
      - run: echo "Additional checks here"
```

