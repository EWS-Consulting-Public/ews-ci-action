# Quick Start

Get started with `ews-ci-action` in 2 minutes.

## Prerequisites

- EWS Python package with `pyproject.toml`
- Repository has `EWS_CREDENTIALS` secret (JSON object with GitLab tokens)
- Package follows standard structure (src/, tests/, etc.)
- Optional: `EWS_CREDENTIALS_KEYS` variable for validation

**Setup credentials:**

```bash
# Set both secret and keys variable
uv run ews-github export-credentials --repo <your-repo> --set-secret --set-keys
```

## Step 1: Create CI Workflow

Create `.github/workflows/ci.yml`:

```yaml
name: CI

on:
  push:
    branches: [main]
    tags: ["v*"]
  pull_request:

jobs:
  ci:
    uses: EWS-Consulting-Public/ews-ci-action/.github/workflows/ci.yml@v1
    with:
      ews-credentials-keys: ${{ vars.EWS_CREDENTIALS_KEYS }} # Optional
    secrets:
      EWS_CREDENTIALS: ${{ secrets.EWS_CREDENTIALS }} # Required
```

## Step 2: Create Release Workflow

Create `.github/workflows/release.yml`:

```yaml
name: Release

on:
  workflow_run:
    workflows: ["CI"]
    types: [completed]
    branches: ["v*"]

permissions:
  contents: write # Required for GitHub releases
  actions: read # Required to download CI artifacts

jobs:
  release:
    uses: EWS-Consulting-Public/ews-ci-action/.github/workflows/release.yml@v1
    with:
      ews-credentials-keys: ${{ vars.EWS_CREDENTIALS_KEYS }} # Optional
    secrets:
      EWS_CREDENTIALS: ${{ secrets.EWS_CREDENTIALS }} # Required
```

## Step 3: Commit and Push

```bash
git add .github/workflows/
git commit -m "ci: use ews-ci-action for CI/CD"
git push
```

## Step 4: Test

Trigger CI by pushing to main or opening a PR:

```bash
git commit --allow-empty -m "test: trigger CI"
git push
```

Check GitHub Actions tab to see it running!

## Step 5: Release

When ready to publish:

```bash
make release  # Or: nox -s release
```

This will:

1. Bump version (CalVer)
2. Create tag
3. Push → triggers CI
4. CI passes → triggers Release
5. Package published to GitLab + GitHub

## What You Get

✅ Automated linting (ruff)  
✅ Automated testing (pytest)  
✅ Automated builds (uv build)  
✅ Automated publishing (GitLab + GitHub)  
✅ No more duplicated workflow files!

## Next Steps

- Customize Python version: Add `with: { python-version: "3.13" }`
- Add custom steps: See [`examples/custom-steps.yml`](examples/custom-steps.yml)
- Test multiple versions: See [`examples/matrix-testing.yml`](examples/matrix-testing.yml)

## Troubleshooting

**CI fails with "401 Unauthorized" or "Missing credentials":**

Ensure `EWS_CREDENTIALS` secret is set:

```bash
uv run ews-github export-credentials --repo <your-repo> --set-secret
```

**CI fails with "Missing required keys in EWS_CREDENTIALS":**

Ensure all required keys are present:

```bash
# Check what's expected
uv run ews-github audit --repo <your-repo>

# Re-export credentials
uv run ews-github export-credentials --repo <your-repo> --set-secret --set-keys
```

**Release doesn't trigger:**

- Verify tag pattern is `v*` (e.g., `v20251118.0`)
- Check that CI workflow completed successfully

**Want to test without publishing?**

- Just use the CI workflow, skip the release workflow
- Or set `upload-artifacts: false` in CI
