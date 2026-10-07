---
status: as-built
covers: why the release workflow declares overlay-named secrets beside EWS_CREDENTIALS, and how its end-to-end step uses them
last-verified: 2026-10-07
---

# ADR 0007: The release end-to-end step takes overlay-named secrets

**Date:** 2026-10-07
**Status:** accepted. Amends [ADR 0002](0002-one-credentials-secret-not-many.md)
for the release workflow only.

## Context

A package that runs work in the cloud may prove one real case when it
releases: a command, supplied by that package, that runs after the release is
published. That command needs a credential that belongs to that one
repository, such as a service-account key allowed to submit jobs for that
study and nothing else.

`EWS_CREDENTIALS` is the wrong home for it. The same object is written onto
every consuming repository, so a key inside it would reach every package whose
tests only install a wheel. ADR 0002 traded least privilege for a single
secret; this credential is the case where that trade does not hold.

A reusable workflow sees only the secrets its caller passes. A caller passes
a secret by name only when the workflow declares it. `secrets: inherit`
passes everything, but GitHub honours it only for a workflow in the same
organization or enterprise as the caller, and this action is called from
other organizations. So "export every secret whose name starts with `EWS_`"
would iterate the declared secrets and nothing else; declaring each overlay
by name says the same thing plainly.

## Decision

- `release.yml` declares two optional secrets, `EWS_GCP__DEFAULT_KEY` and
  `EWS_GCP__PROJECT`, beside `EWS_CREDENTIALS`, and one optional input,
  `e2e-command`.
- When `e2e-command` is set, a last step runs it. The command reaches the
  shell through `env:`, never through `${{ }}` in `run:`.
- That step, and no other, receives the two secrets in its environment. A
  secret the caller did not pass is unset before the command runs, and the
  step logs each name as `set` or `not passed`. Values are never printed.
- The names are the EWS configuration layer's environment overlay,
  `EWS_<SECTION>__<FIELD>`. The overlay wins over every configuration file and
  over the flat names exported from `EWS_CREDENTIALS`, so a library that reads
  its configuration through that layer resolves them with no code of its own.
- A study that needs another overlay adds one declared secret here, the way
  `EWS_CREDENTIALS` is declared.

## Consequences

- **The key reaches one step of one repository's release.** The build,
  publish and GitHub-release steps never see it, and neither does `ci.yml`.
- **An empty overlay is not harmless, which is why the step unsets it.** The
  configuration layer treats an exported empty overlay as a value: an empty
  `EWS_GCP__DEFAULT_KEY` would blank the key that `EWS_CREDENTIALS` supplies
  under its flat name. GitHub expands a secret the caller did not pass to an
  empty string.
- **A command that fails marks the run red after the release is published.**
  The wheel and the GitHub release stand; the end-to-end result is evidence
  about that release, not a gate on it.
- **Adding an overlay is a change here and in each caller that needs it**, the
  cost ADR 0002 avoided. It is paid per overlay a study needs, not per
  credential the callers share.
- **Callers that set neither the input nor the secrets see no change.** The
  step is skipped.

## Related

- [ADR 0002](0002-one-credentials-secret-not-many.md): the single secret this
  amends
- [../workflows.md](../workflows.md) § `release.yml`: the input and the
  secrets
- [../credentials.md](../credentials.md): where each credential lands
