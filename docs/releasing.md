---
status: as-built
covers: changing this repository, testing a change, and moving the version tags
last-verified: 2026-08-07
---

# Changing and releasing this repository

This repository has no build and no test suite. Its "release" is a git tag,
because that is how GitHub Actions resolves a `uses:` reference
([ADR 0004](decisions/0004-v1-is-a-moving-major-tag.md)).

## Testing a change

There is no way to run these workflows locally. The loop is:

1. Push the change to a branch.
2. In a consuming repository, point at the branch instead of the tag:

   ```yaml
   uses: EWS-Consulting-Public/ews-ci-action/.github/workflows/ci.yml@my-branch
   ```

   A `uses:` reference takes any ref — branch, tag or SHA.

3. Trigger that repository's workflow and read the run.
4. Revert the consumer to `@v1` before merging.

Test against **more than one** consuming repository before moving `v1`. The tag
is shared, so a regression reaches every consumer at once.

## Tagging

```bash
git tag -a v1.2.0 -m "Add pytest-args input"
git push origin v1.2.0

# move the major pointer
git tag -fa v1 -m "Update v1 to v1.2.0"
git push --force origin v1
```

A breaking change gets a new major instead, and `v1` stays where it is:

```bash
git tag -a v2.0.0 -m "Breaking: <what breaks>"
git push origin v2.0.0
git tag -a v2 -m "Create v2"
git push origin v2
```

Force-pushing `v1` is deliberate and is the only place in this repository where
that is correct.

**Current state (2026-09-05):** `v1` was moved to `33e94f6` on 2026-09-05 (the
credential export and `data-plan:` change); `v1.0` still marks `952e37c`. Both
are lightweight and
not annotated. The `-a` form above creates annotated tags; the existing
ones are not.

## What counts as breaking

The public surface is the inputs of the two workflows and the composite action,
plus what a consumer must have on disk.

Breaking:

- Removing or renaming an input.
- Changing a default in a way that changes behaviour for a caller that did not
  set it.
- Requiring something new of the consuming repository — a new nox session, a
  new credential key, a different runner.
- Changing where the composite action writes a file.

Not breaking:

- Adding an optional input with a default that preserves current behaviour.
- Bumping a third-party action to a compatible version.

## Changing inputs

1. Add to the `inputs:` block with a description and a default.
2. Thread it through every layer that needs it — the workflow input, the
   `uses:` call sites, the composite action's input, the step environment.
   `ews-credentials-keys` is threaded through all four and read by none; that
   is the failure mode to avoid.
3. Update [workflows.md](workflows.md) or [action-reference.md](action-reference.md).
4. Add an example under `examples/` if it changes how a caller is written.

## Shell safety

Anything derived from a commit message, a tag, a branch name or a PR title is
attacker-controlled. Pass it through `env:` and reference it as a shell
variable — never interpolate `${{ }}` into a `run:` block. The `check-skip` job's `COMMIT_MSG` block is
the pattern; it exists because the alternative was a shell-injection bug
(commit `952e37c`).

## Documentation

`docs/` follows a fixed shape: a `docs/README.md` map, per-file YAML
`status:` / `covers:` / `last-verified:` headers, and ADRs under
`docs/decisions/`. Update `last-verified:` when you re-check a page against the
YAML.

`.cursor/` is authored and `.claude/` is generated:

```bash
uv run --no-project python scripts/sync_agent_config.py          # regenerate
uv run --no-project python scripts/sync_agent_config.py --check  # verify
```

`--no-project` because there is no `pyproject.toml` here. `prek run -a` runs
the `--check` form, so an unsynced `.cursor/` edit fails the commit.

## Questions

Open an issue on this repository — but keep in mind it is **public**. Anything
naming internal infrastructure belongs somewhere else. See
`.cursor/rules/public-repo-boundary.mdc`.
