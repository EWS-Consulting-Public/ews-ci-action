---
status: as-built
covers: why end-to-end runs are a reusable workflow the consumer triggers, and why it declares overlay-named secrets beside EWS_CREDENTIALS
last-verified: 2026-10-07
---

# ADR 0007: End-to-end runs are a reusable workflow the consumer triggers

**Date:** 2026-10-07
**Status:** accepted. Amends [ADR 0002](0002-one-credentials-secret-not-many.md)
for the end-to-end workflow only.

## Context

A package that runs work in the cloud may prove one real case: a command,
supplied by that package, that submits a job and reads back what it produced.
When that runs is the package's call. One package wants it after each
release, another on demand, another when a commit asks for it.

That command needs a credential that belongs to that one repository, such as
a service-account key allowed to submit jobs for that study and nothing else.
`EWS_CREDENTIALS` is the wrong home for it. The same object is written onto
every consuming repository, so a key inside it would reach every package whose
tests only install a wheel. ADR 0002 traded least privilege for a single
secret; this credential is the case where that trade does not hold.

A reusable workflow sees only the secrets its caller passes. A caller passes
a secret by name only when the workflow declares it. `secrets: inherit`
passes everything, but GitHub honours it only for a workflow in the same
organization or enterprise as the caller, and this action is called from
other organizations.

## Decision

- `e2e.yml` is its own reusable workflow: check out a ref, run the setup
  action, run one command. It has no trigger of its own. The consumer's
  caller owns the `on:` block, so the consumer decides when it runs.
- The command is a required input, `command`. It reaches the shell through
  `env:`, never through `${{ }}` in `run:`.
- It declares two optional secrets, `EWS_GCP__DEFAULT_KEY` and
  `EWS_GCP__PROJECT`, beside an optional `EWS_CREDENTIALS`. Only the command's
  step receives the two overlays. A secret the caller did not pass is unset
  before the command runs, and the step logs each name as `set` or
  `not passed`. Values are never printed.
- The names are the EWS configuration layer's environment overlay,
  `EWS_<SECTION>__<FIELD>`. The overlay wins over every configuration file and
  over the flat names exported from `EWS_CREDENTIALS`, so a library that reads
  its configuration through that layer resolves them with no code of its own.
- A study that needs another overlay adds one declared secret here, the way
  `EWS_CREDENTIALS` is declared.
- `ci.yml` and `release.yml` do not change.

## Consequences

- **The trigger is the consumer's.** A manual `workflow_dispatch`, a push
  whose commit message carries a marker, the completion of the release
  workflow: each is a line in the consumer's caller, and none needs a change
  here. `examples/e2e.yml` shows all three.
- **The key reaches one step of one repository's run.** Build, publish and
  test steps never see it, and neither do `ci.yml` and `release.yml`.
- **An empty overlay is not harmless, which is why the step unsets it.** The
  configuration layer treats an exported empty overlay as a value: an empty
  `EWS_GCP__DEFAULT_KEY` would blank the key that `EWS_CREDENTIALS` supplies
  under its flat name. GitHub expands a secret the caller did not pass to an
  empty string.
- **A run never undoes or blocks a release.** It is evidence about the commit
  or tag it checked out.
- **Adding an overlay is a change here and in each caller that needs it**, the
  cost ADR 0002 avoided. It is paid per overlay a study needs, not per
  credential the callers share.
- **`e2e-selftest.yml` proves the mechanics on this repository**, with no
  credential: one overlay passed, one not, and a command that checks the
  first is set and the second unset.

## Related

- [ADR 0002](0002-one-credentials-secret-not-many.md): the single secret this
  amends
- [../workflows.md](../workflows.md) § `e2e.yml`: the inputs, the secrets and
  the step
- [../credentials.md](../credentials.md): where each credential lands
