---
name: adopt-ews-ci
description: Put the ews-ci-action reusable workflows into a Python package repository, or fix one whose CI is failing after adopting them - "add this action to my repo's CI", "wire up CI for this package", "migrate this repo to ews-ci-action", "why is CI failing with 401 / no EWS_CREDENTIALS", "the release workflow never runs", "add an end-to-end run", "call e2e.yml". Covers the prerequisite check, the CI, release and end-to-end (e2e.yml) caller workflows, the permissions and secrets that reusable workflows do NOT inherit, and the failure modes that look like something else.
---

# Adopting `ews-ci-action` in a package repository

This procedure runs **in the consuming repository**, not in `ews-ci-action`.
It writes two workflow files there, three with the end-to-end caller, and
changes nothing here.

## Locked conventions — do not re-open

- **Reference `@v1`**, never a branch and never a SHA, except while testing a
  change to the action itself. `v1` is force-moved forward on purpose
  (ADR 0004).
- **Credentials are one secret**, `EWS_CREDENTIALS`, a JSON object. Do not
  create per-credential secrets (ADR 0002). The one exception is the
  end-to-end overlays `e2e.yml` declares by name, which belong to one
  repository (ADR 0007).
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
    branches: ["v*"]

permissions:
  contents: write
  actions: read

jobs:
  release:
    uses: EWS-Consulting-Public/ews-ci-action/.github/workflows/release.yml@v1
    secrets:
      EWS_CREDENTIALS: ${{ secrets.EWS_CREDENTIALS }}
```

Two things here are load-bearing and are the usual cause of "the release
never runs":

- `workflows: ["CI"]` matches the CI workflow's **`name:` field**, not its
  filename.
- The `permissions` block is **required**: the reusable workflow does not
  request permissions on the caller's behalf.

**Keep `branches: ["v*"]`.** GitHub matches it against the triggering CI run's
`head_branch`, the tag name on a tag run, so the release workflow starts only
for tags. Without it every branch and pull-request CI run also starts a release
run whose job is skipped.

## 4. Write the end-to-end caller

Skip this step unless the package proves one real run end to end. `e2e.yml`
runs one command the caller supplies and has no trigger of its own: the
caller's `on:` block decides when it runs.

`.github/workflows/e2e.yml`:

```yaml
name: End-to-end

on:
  workflow_dispatch:
  push:
    branches: [main]

jobs:
  e2e:
    if: >-
      github.event_name == 'workflow_dispatch' ||
      (github.event_name == 'push' && contains(github.event.head_commit.message, '[e2e]'))
    uses: EWS-Consulting-Public/ews-ci-action/.github/workflows/e2e.yml@v1
    with:
      command: uv run <your-cli> <your-end-to-end-command>
    secrets:
      EWS_CREDENTIALS: ${{ secrets.EWS_CREDENTIALS }}
      EWS_GCP__DEFAULT_KEY: ${{ secrets.EWS_GCP__DEFAULT_KEY }}
      EWS_GCP__PROJECT: ${{ secrets.EWS_GCP__PROJECT }}
```

- **`command` is the one required input.** It reaches the shell through
  `env:`. Never build it from a commit message, branch name or PR title.
- **Forward each secret by name**, never `secrets: inherit`, which GitHub
  honours only from the same organization or enterprise as this action. All
  three are optional; drop the lines the command does not need.
- **The two `EWS_GCP__*` overlays are repository secrets of this repository**,
  never keys of `EWS_CREDENTIALS`. Only the command's step sees them, and one
  the caller did not pass is unset, not exported empty.
- **Nothing is installed before the command.** A `uv run` inside it syncs the
  project.
- The commit message is read in `if:` only, never in a shell.
- `examples/e2e.yml` in `ews-ci-action` also shows an after-release trigger. It
  does not fire as written (`docs/README.md` § *Open questions* there); leave
  it out.

## 5. Remove what is now duplicated

Delete the lint / test / build / publish jobs the two callers replace. Keep any
job that does something these workflows do not — docs builds, notebook checks,
deployment. Chain it with `needs: ci`.

## 6. Verify

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

For the end-to-end caller, start it once by hand and read its log:

```bash
gh workflow run e2e.yml --repo <owner>/<name>
gh run list --repo <owner>/<name> --workflow e2e.yml --limit 1
```

The `End-to-end` step logs `<NAME>: set` for each overlay forwarded and
`<NAME>: not passed` for each one left out, then runs the command.

## The failure modes

| Symptom | Cause |
| --- | --- |
| `⚠️ No EWS_CREDENTIALS provided` although the secret is set | The caller did not forward it — reusable workflows do not inherit secrets. Add the `secrets:` block, or `secrets: inherit`. Same message if the secret's value is not valid JSON, because every `jq` extraction then yields empty |
| `401` / unresolvable dependency in `uv sync` | `gitlab_api_read_token` and `gitlab_package_registry_url` must **both** be in the JSON; the action needs the pair and skips the registry silently if either is absent |
| Release job never starts | CI did not run on the tag, CI was not green, `workflows:` does not match the CI workflow's `name:`, or the tag does not start with `v` |
| Release fails at "Build package" | No `nox -s build` session. `release.yml` calls it unconditionally, regardless of `use-nox-build` |
| `EWS_GCP__…: not passed` although the secret is set | The end-to-end caller did not forward it by name in its `secrets:` block |
| The end-to-end caller cannot find `e2e.yml@v1` | `v1` points at a release older than `e2e.yml`; `git ls-remote --tags https://github.com/EWS-Consulting-Public/ews-ci-action` shows where. Moving `v1` is Fabien's call; do not pin a branch to work around it |

## Do not

- **Do not edit `ews-ci-action` to accommodate one repository.** An input only
  one consumer would use does not belong in a shared action. Use the composite
  action directly in a custom job instead.
- **Do not pin to a branch or a SHA** in a normal adoption. `@v1` is the
  contract.
- **Do not add credentials as separate repository secrets.** One JSON secret
  (ADR 0002); the `e2e.yml` overlays are the declared exception (ADR 0007).
- **Do not copy a credential value** into a workflow file, a commit message or
  an issue — and never into `ews-ci-action`, which is public.
- **Do not paste internal hostnames, registry URLs or paths** into anything
  that ends up in `ews-ci-action`. The registry URL arrives at run time inside
  the secret precisely so it is not written down.
- **Do not move the `v1` tag** to ship a fix you needed for one repository.
  That deploys to every consumer at once and is Fabien's call.
