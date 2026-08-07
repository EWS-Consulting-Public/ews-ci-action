# AGENTS.md — ews-ci-action

Reusable GitHub Actions CI/CD for uv-based Python packages: one composite
action that provisions a runner, and two `workflow_call` workflows that lint,
test, build and release the packages that call them.

> ## ⚠ THIS REPOSITORY IS PUBLIC
>
> `EWS-Consulting-Public/ews-ci-action` is world-readable, and so is its commit
> history and the edit history of every issue. **The habits that are safe in a
> private repository are not safe here.** Nothing internal goes in here — no
> server or host names, no share layouts or drive paths, no local checkout
> paths, no registry ids or accounts, no client names, nothing about how EWS's
> private repositories are organised or worked on.
> Credential **key names** are the action's public interface and are fine; a
> credential **value** never is. Read `.cursor/rules/public-repo-boundary.mdc`
> before writing anything. It is the first rule for a reason.

**This is not a Python package.** No `pyproject.toml`, no `src/`, no tests. It
ships YAML that GitHub executes on *other* repositories, so the subject of the
documentation is a contract — what a consumer must provide, what lands on the
runner, and what silently does nothing — not an API.

## Layout

```text
.github/actions/setup-ews-ci/action.yml   the composite action - uv, Python, credential files
.github/workflows/ci.yml                  reusable: check-skip -> lint -> test (matrix) -> build
.github/workflows/release.yml             reusable: on a green tagged CI run, build, publish, release
examples/                                 three copy-paste caller workflows
docs/                                     documentation map + ADRs
docs/_old/                                pre-2026-08-07 docs, parked pending review - do not cite
scripts/sync_agent_config.py              generates .claude/ from .cursor/
.cursor/ .claude/                         rules (.cursor authored, .claude generated)
```

`LICENSE`, `examples/*.yml` and everything under `.github/` are **the product**.
Documentation changes do not touch them.

## Commands

There is no build, no test suite and no way to run these workflows locally.
The only executable thing in the repository is the config generator:

```bash
uv run --no-project python scripts/sync_agent_config.py          # regenerate .claude/
uv run --no-project python scripts/sync_agent_config.py --check  # verify; exit 1 on drift
prek run -a                                                      # the gate - run before pushing
```

**`--no-project` is required.** There is no `pyproject.toml`, so plain
`uv run` fails looking for one. The script is stdlib-only, so no environment is
needed. Never bare `python` on the Windows host — it is the Store stub.

Verifying a change means pushing a branch and pointing a consuming repository
at `@my-branch` instead of `@v1`. See [`docs/releasing.md`](docs/releasing.md).

## Read before editing

1. [`docs/README.md`](docs/README.md) — the documentation map, the flow
   diagram, the status legend, and § *Open questions* (five things the YAML
   does not settle — check there before concluding something is a bug).
2. [`docs/decisions/`](docs/decisions/) — five ADRs. If you are about to
   propose splitting `EWS_CREDENTIALS` into separate secrets, converting the
   composite action to JavaScript, having the release job download CI's
   artifact, or pinning consumers to exact tags — that argument is already
   recorded. Read the ADR before reopening it.
3. Always-on rules `public-repo-boundary` (what may never be written here) and
   `ci-action-project` (the scope, the consumer contract, the do-nots).

Reference pages: [`action-reference.md`](docs/action-reference.md) (every file
the action writes), [`workflows.md`](docs/workflows.md) (every input),
[`credentials.md`](docs/credentials.md), [`adopting.md`](docs/adopting.md),
[`releasing.md`](docs/releasing.md).

## Hard boundaries

- **Never write internal detail into this repository.** See the banner and
  rule `public-repo-boundary`. This includes commit messages, issue text and
  code comments — all world-readable, all permanent.
- **Never copy a credential value** anywhere. Key names only.
- **Never interpolate `${{ }}` into a `run:` block** when the value is
  attacker-controlled — a commit message, tag, branch name or PR title. Pass it
  through `env:` and reference the shell variable. `ci.yml:74-77` is the
  pattern, and it exists because the alternative was a shell-injection bug
  (`952e37c`).
- **Never move the `v1` tag** as part of a routine change. It is force-pushed
  and reaches every consuming repository at once, untested
  ([ADR 0004](docs/decisions/0004-v1-is-a-moving-major-tag.md)). Tagging is
  Fabien's call.
- **Never remove or rename a workflow input.** Consumers on `@v1` break
  immediately and silently. Add optional inputs with behaviour-preserving
  defaults instead.
- **Never edit `.claude/` by hand** — generated from `.cursor/`.
- **Do not verify a claim from `docs/_old/`.** Eleven of its statements are
  wrong against the YAML; the list is in
  [`docs/_old/README.md`](docs/_old/README.md).

## Non-goals

Not a Python package and not a place for Python code. Not a general-purpose
Actions library — it serves uv-based EWS packages publishing to one private
registry, and an input that only one consumer would use does not belong here.
Not a credential store: it consumes a secret someone else provisions. Not
cross-platform — every step is `shell: bash` on `ubuntu-latest`, and nothing
here has been exercised on Windows or macOS runners.

## The hub

Wider context for this repository — checkout facts, gates, do-nots and the
dated finding trail — lives in Fabien's private context repository, not here.
A session dispatched into this repository is briefed from there.

**This repository has no tracking issue for that trail, deliberately.** It
would be world-readable. Do not create one.

A repository instrumented this way normally links, from this section, to the
per-repository briefing file that a dispatching session loads. **That link is
omitted here on purpose, because this repository is public** — the path would
itself disclose how the private side is organised. The omission is a decision,
not an oversight, and it is the rule in
`.cursor/rules/public-repo-boundary.mdc` applied to this file. A session
dispatched here is given that context in its brief instead.
