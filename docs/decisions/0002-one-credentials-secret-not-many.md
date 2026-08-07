---
status: as-built
covers: why credentials arrive as one JSON secret rather than N repository secrets
last-verified: 2026-08-07
---

# ADR 0002 — All credentials arrive as one JSON secret

**Date:** recorded 2026-08-07 from `action.yml` and both workflows
**Status:** accepted

## Context

The setup step needs up to five values: two registry tokens, a registry URL,
and two data-source passwords. The obvious GitHub-native shape is five
repository secrets, each declared as a `secrets:` input on the reusable
workflow and forwarded individually.

That shape has a specific cost in *reusable* workflows. A reusable workflow
must declare every secret it accepts, and every caller must forward every one
of them by name. Adding a sixth credential is then a change in three places per
consuming repository — the workflow's `secrets:` block, the caller's forwarding
block, and the repository's secret store — and every existing consumer's
workflow file has to be edited before the new credential reaches them.

## Decision

**One repository secret, `EWS_CREDENTIALS`, holding a JSON object.** The
composite action parses it with `jq` and extracts each key by name
(`action.yml:70-88`).

A companion repository *variable*, `EWS_CREDENTIALS_KEYS`, was intended to
carry the list of keys that ought to be present, so the action could validate
the secret's shape.

## Consequences

- **Adding a credential does not touch any consumer's YAML.** A new key in the
  JSON plus a new extraction line in `action.yml` reaches every repository on
  `@v1` at once. This is the property being bought.
- **The workflow interface stays at one secret.** `ci.yml:61-64` and
  `release.yml:21-24` each declare exactly one, and a caller forwards one line.
- **The secret is all-or-nothing.** GitHub secrets have no sub-scoping, so a
  workflow that only needs to *install* private wheels still receives the
  publish token and both data-source passwords. There is no least-privilege
  path short of a second secret, which is what this decision traded away.
- **A malformed secret is indistinguishable from a missing one.**
  `jq -r '.key // empty'` on invalid JSON yields empty for every key, so the
  action takes the same "no credentials" branch and continues. There is no
  parse check and no failure.
- **GitHub's log masking is applied to the whole JSON blob**, which it will
  redact if it appears verbatim. The individual values parsed out of it by `jq`
  are separate strings; do not add a step that echoes one.
- **The intended validation was never implemented.** `EWS_CREDENTIALS_KEYS` is
  declared as an input on the composite action (`action.yml:17-20`), exported
  into the setup step's environment (`action.yml:38`), declared on both
  workflows and forwarded at four call sites — and read by nothing.
  `grep -rn 'EWS_CREDENTIALS_KEYS' .github/` (2026-08-07) returns only the
  declaration and the forwarding. So the mechanism that was supposed to
  compensate for the loss of per-secret typing does not exist, and a key
  missing from the JSON is discovered as a `401` in `uv sync`.

## Evidence

`action.yml:74-78` — five `jq -r '.<key> // empty'` extractions from
`$EWS_CREDENTIALS`.

`action.yml:81-88` — the else branch sets all five to the empty string and
prints `⚠️ No EWS_CREDENTIALS provided`.

`action.yml:93`, `:123`, `:143`, `:155` — each downstream file is written only
when its inputs are non-empty; none is required.

## Related

- [../credentials.md](../credentials.md) — the keys and where each one lands
- [../README.md](../README.md) § *Open questions* — the unread variable
- [ADR 0001](0001-composite-action-not-javascript.md) — why the parsing is
  `jq` in bash
