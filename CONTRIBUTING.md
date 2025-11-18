# Contributing to ews-ci-action

Thank you for contributing to the EWS CI/CD infrastructure!

## Making Changes

### Testing Your Changes

**Before releasing:**

1. Create a feature branch:
   ```bash
   git checkout -b feature/my-improvement
   git push -u origin feature/my-improvement
   ```

2. Update a test package (e.g., `ews-jupyter`) to use your branch:
   ```yaml
   # In ews-jupyter/.github/workflows/ci.yml
   jobs:
     ci:
       uses: EWS-Consulting-Private/ews-ci-action/.github/workflows/ci.yml@feature/my-improvement
   ```

3. Trigger the workflow and verify behavior

4. Once verified, merge to `main` and update version tags

### Releasing a New Version

We use Git tags to version this action (following GitHub Actions conventions).

**For patch releases (v1.0.1):**

```bash
git tag -a v1.0.1 -m "Fix: description of fix"
git push origin v1.0.1

# Update major version pointer
git tag -fa v1 -m "Update v1 to v1.0.1"
git push --force origin v1
```

**For minor releases (v1.1.0):**

```bash
git tag -a v1.1.0 -m "Feature: description of new feature"
git push origin v1.1.0

# Update major version pointer
git tag -fa v1 -m "Update v1 to v1.1.0"
git push --force origin v1
```

**For major releases (v2.0.0):**

Breaking changes require a new major version:

```bash
git tag -a v2.0.0 -m "Breaking: description of breaking change"
git push origin v2.0.0

# Create new major version pointer
git tag -a v2 -m "Create v2 tag"
git push origin v2
```

**Why we tag this way:**

- Consumers use `@v1` to automatically get the latest v1.x.x
- Specific versions like `@v1.2.3` are available for pinning
- Moving `v1` tag forward maintains compatibility while delivering fixes

## What to Change

### Adding New Inputs

When adding inputs to workflows or actions:

1. Add to the `inputs:` section with description and default
2. Update `README.md` with the new input
3. Add an example to `examples/` showing its use
4. Consider backward compatibility (new inputs should be optional)

### Modifying Workflow Logic

For changes to `ci.yml` or `release.yml`:

1. Ensure backward compatibility (don't break existing consumers)
2. Test with at least 2 different EWS packages
3. Update documentation if behavior changes

### Adding New Workflows

When adding a new reusable workflow:

1. Create in `.github/workflows/`
2. Document inputs/outputs in comments
3. Add to `README.md` under "Available Workflows"
4. Create example in `examples/`

## Code Style

- Use clear, descriptive names for jobs and steps
- Add comments explaining non-obvious logic
- Keep workflows DRY (don't repeat yourself)
- Use defaults that work for 80% of cases

## Questions?

Ask in EWS-Consulting-Private discussions or open an issue.

