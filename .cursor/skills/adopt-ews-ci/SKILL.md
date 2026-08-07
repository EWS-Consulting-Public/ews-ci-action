---
name: adopt-ews-ci
description: Put the ews-ci-action reusable workflows into a Python package repository, or fix one whose CI is failing after adopting them - "add this action to my repo's CI", "wire up CI for this package", "migrate this repo to ews-ci-action", "why is CI failing with 401 / no EWS_CREDENTIALS", "the release workflow never runs". Covers the prerequisite check, both caller workflows, the permissions and secrets that reusable workflows do NOT inherit, and the four failure modes that look like something else.
---

# Adopting `ews-ci-action` in a package repository

This procedure runs **in the consuming repository**, not in `ews-ci-action`.
It writes two workflow files there and changes nothing here.

## Locked conventions — do not re-open

- **Reference `@v1`**, never a branch and never a SHA, except while testing a
  change to the action itself. `v1` is force-moved forward on purpose
  (ADR 0004).
- **Credentials are one secret**, `EWS_CREDENTIALS`, a JSON object. Do not
  create per-credential secrets (ADR 0002).
- **Runners are `ubuntu-latest`.** The composite action is `shell: bash` with
  POSIX paths; there is no OS matrix and adding one is not a small change.
- Docs for every input: `docs/workflows.md` in `ews-ci-action`.

## 1. Check the prerequisites

In the consuming repository:

```bash
test -f pyproject.toml && echo "pyproject: ok"
test -f uv.lock && echo "lockfile: committed (uv sync --frozen)" || echo "lockfile: none (locked at run time)"
grep -n 'def build\|"build"\|session(name="build")' noxfile.py 2>/dev/null || echo "NO nox build session"
grep -n 'def publish\|"publish"\|session(name="publish")' noxfile.py 2>/dev/null || echo "NO nox publish session"
```

- **No `nox -s build`** → either add one, or set `use-nox-build: false` for CI.
  Note that `release.yml` calls `nox -s build` *unconditionally*, so a
  repository that releases needs the session either way.
- **No `nox -s publish`** → set `use-nox-publish: false` to use the inline
  `twine` path instead.

Confirm the secret exists (needs repo admin):

```bash
gh secret list --repo <owner>/<name> | grep EWS_CREDENTIALS
```

If it is missing, it is provisioned by the EWS credential tooling
(`ews-github-utils`), not pasted by hand. Do not invent its contents.

## 2. Write the CI caller

`.github/workflows/ci.yml` in the consuming repository:

```yaml
name: CI

on:
  push:
    branches: [main]
    tags: ["v*"]
  pull_request:
    branches: [main]

jobs:
  ci:
    uses: EWS-Consulting-Public/ews-ci-action/.github/workflows/ci.yml@v1
    with:
      python-versions: '["3.12", "3.13"]'
    secrets:
      EWS_CREDENTIALS: ${{ secrets.EWS_CREDENTIALS }}
```

**The `tags: ["v*"]` trigger is required** if the repository will release. The
release workflow keys off a successful CI run *on a tag*; a CI that ignores
tags never produces one.

## 3. Write the release caller

Skip this step if the package is not published.

`.github/workflows/release.yml`:

```yaml
name: Release

on:
  workflow_run:
    workflows: ["CI"]
    types: [completed]

permissions:
  contents: write
  actions: read

jobs:
  release:
    uses: EWS-Consulting-Public/ews-ci-action/.github/workflows/release.yml@v1
    secrets:
      EWS_CREDENTIALS: ${{ secrets.EWS_CREDENTIALS }}
```

Three things here are load-bearing and are the usual cause of "the release
never runs":

- `workflows: ["CI"]` matches the CI workflow's **`name:` field**, not its
  filename.
- The `permissions` block is **required** — the reusable workflow does not
  request permissions on the caller's behalf.
- **Do not add a `branches:` filter** to the `workflow_run` trigger. Tag runs
  are not branch runs; the reusable workflow already tests `head_branch` for
  the `v` prefix.

## 4. Remove what is now duplicated

Delete the lint / test / build / publish jobs the two callers replace. Keep any
job that does something these workflows do not — docs builds, notebook checks,
deployment. Chain it with `needs: ci`.

## 5. Verify

```bash
git add .github/workflows/ && git commit -m "ci: use ews-ci-action" && git push
gh run list --repo <owner>/<name> --limit 3
gh run view <run-id> --log-failed
```

The run must show `check-skip`, `lint`, `test` (one per Python version) and
`build`, with `build` starting only after lint and test finish. In the setup
step's log, confirm:

```text
✅ Configured uv index: gitlab (…)
```

If instead it says `ℹ️ No GitLab read token or package registry URL, skipping
package registry setup`, the registry was not configured and any private
dependency will fail to resolve.

Then test the release path with a real tag, and check that the GitHub release
carries the wheel and `uv.lock`.

## The four failure modes

| Symptom | Cause |
| --- | --- |
| `⚠️ No EWS_CREDENTIALS provided` although the secret is set | The caller did not forward it — reusable workflows do not inherit secrets. Add the `secrets:` block, or `secrets: inherit`. Same message if the secret's value is not valid JSON, because every `jq` extraction then yields empty |
| `401` / unresolvable dependency in `uv sync` | `gitlab_api_read_token` and `gitlab_package_registry_url` must **both** be in the JSON; the action needs the pair and skips the registry silently if either is absent |
| Release job never starts | CI did not run on the tag, CI was not green, `workflows:` does not match the CI workflow's `name:`, or a `branches:` filter was added to `workflow_run` |
| Release fails at "Build package" | No `nox -s build` session. `release.yml` calls it unconditionally, regardless of `use-nox-build` |

## Do not

- **Do not edit `ews-ci-action` to accommodate one repository.** An input only
  one consumer would use does not belong in a shared action. Use the composite
  action directly in a custom job instead.
- **Do not pin to a branch or a SHA** in a normal adoption. `@v1` is the
  contract.
- **Do not add credentials as separate repository secrets.** One JSON secret
  (ADR 0002).
- **Do not copy a credential value** into a workflow file, a commit message or
  an issue — and never into `ews-ci-action`, which is public.
- **Do not paste internal hostnames, registry URLs or paths** into anything
  that ends up in `ews-ci-action`. The registry URL arrives at run time inside
  the secret precisely so it is not written down.
- **Do not move the `v1` tag** to ship a fix you needed for one repository.
  That deploys to every consumer at once and is Fabien's call.
