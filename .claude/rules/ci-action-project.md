<!-- GENERATED FROM .cursor/rules/ci-action-project.mdc BY scripts/sync_agent_config.py.
     Edit the .cursor source, then run: uv run python scripts/sync_agent_config.py -->

**Applies to:** always - this rule is in force for every change.

# ews-ci-action — the boundary

One composite action and two `workflow_call` workflows for uv-based Python
packages that publish to a private GitLab package registry. That is the whole
job.

**It is not a Python package.** No `pyproject.toml`, no `src/`, no tests.
Nothing here executes except on a GitHub runner, in someone else's repository.

**Every consumer references `@v1`, a tag that is force-moved forward.** A
change merged and tagged reaches every consuming repository on its next run,
with no PR and no review anywhere else. Treat every edit as a production
deployment to all consumers simultaneously
(ADR 0004).

## The consumer contract

What a consuming repository must have, and what breaking it costs:

- `pyproject.toml`, uv-resolvable. A committed `uv.lock` triggers
  `uv sync --frozen --all-extras`.
- `ruff` and `pytest` runnable via `uv run`.
- A `nox` session named `build` — required by `release.yml` unconditionally,
  even if the repository set `use-nox-build: false` for CI.
- A `nox` session named `publish` when releasing with the default.
- An `EWS_CREDENTIALS` secret, explicitly forwarded. Reusable workflows do not
  inherit secrets.
- `permissions: contents: write` and `actions: read` on the release caller.
  The reusable workflow does not request them.

Adding to that list is a **breaking change**, even when no input changed.

## Never do these

- **Never remove or rename an input** of a workflow or the composite action.
  Consumers on `@v1` break immediately, and nothing warns them. Add optional
  inputs with defaults that preserve current behaviour.
- **Never change a default** in a way that changes behaviour for a caller who
  did not set it. That is the same breakage with no diff in the caller's repo.
- **Never move the `v1` tag** as part of a change. Tagging is a deliberate,
  separate act and it is Fabien's call. Test on a branch ref first, against
  more than one consumer.
- **Never interpolate `${{ }}` into a `run:` block** for anything
  attacker-controlled — commit message, tag, branch name, PR title, issue text.
  Pass it through `env:` and use the shell variable. `ci.yml:74-77` is the
  pattern; commit `952e37c` is why.
- **Never echo a parsed credential.** `jq` output is a fresh string and is not
  covered by GitHub's masking of the original secret.
- **Never add a step that fails open on missing credentials** without saying so
  in `docs/action-reference.md` § *Failure modes*. The existing script already
  degrades silently everywhere; do not deepen that without recording it.
- **Never trust `docs/_old/`.** Eleven of its claims are wrong against the
  YAML; the list is in `docs/_old/README.md`.

## Things that look like bugs and are recorded

Check `docs/README.md` § *Open questions* before "fixing" any of these:

- `ews-credentials-keys` is threaded through four layers and read by nothing.
- `run-coverage` defaults `true` while `upload-coverage` defaults `false`.
- `EWS_DATASETS_SYSTEM_CACHE_PATH` is step-scoped and never reaches
  `$GITHUB_ENV`, so later steps do not see it (ADR 0003).
- `check-skip` reads `github.event.head_commit.message`, empty on
  `pull_request`.
- The `~/.config/ews/` flat copy exists "for old tests" that are not named.

They are real, they are written down, and changing any of them ships to every
consumer at once. Propose before acting.

## Tooling

```bash
uv run --no-project python scripts/sync_agent_config.py --check
prek run -a
```

**`--no-project` is required** — there is no `pyproject.toml`, so plain
`uv run` fails looking for one. The generator is stdlib-only. Never bare
`python` on Windows; it is the Store stub.

There is no way to run the workflows locally. Verification is a branch ref in a
consuming repository — `docs/releasing.md` § *Testing a change*.
