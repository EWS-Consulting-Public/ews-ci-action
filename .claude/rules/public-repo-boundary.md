<!-- GENERATED FROM .cursor/rules/public-repo-boundary.mdc BY scripts/sync_agent_config.py.
     Edit the .cursor source, then run: uv run python scripts/sync_agent_config.py -->

**Applies to:** always - this rule is in force for every change.

# This repository is public

`EWS-Consulting-Public/ews-ci-action` is **world-readable**, and it is the only
public repository in the estate. Every other repository you may have worked in
is private; the habits that are safe there are not safe here.

**What is published is not only the working tree.** The commit history, every
diff, every commit message, and the *edit history* of every issue and comment
are all public and all permanent. Deleting a line later does not unpublish it.
There is no undo.

## Never write here

- **Host or server names** — internal machines, dev boxes, file servers, any
  `*.local` or intranet hostname.
- **Share layouts and drive paths** — `F:`, `P:`, `R:`, `X:`, UNC paths, DFS
  namespaces, mount points, directory structures on internal storage.
- **Local checkout paths** — any absolute path on a workstation or dev host,
  Windows or POSIX, and anything naming a person's machine or where a
  repository sits on it.
- **Registry identifiers** — the package registry's project id, its account or
  owner, its URL. The registry URL arrives at run time inside
  `EWS_CREDENTIALS`; that is deliberate, and it is why no URL appears in the
  YAML. Do not "helpfully" document it.
- **Client names, site names, commercial detail.**
- **Estate structure** — board numbers, tracker file paths, the hub repository's
  name, passport names, per-repo issue numbers, how work is coordinated.
- **Any credential value.** Not a token, not a password, not a fragment, not an
  example that looks real. Not in a doc, a test fixture, a commit message or an
  issue.

This applies to **files, commit messages, issue and PR text, and code
comments** equally.

## What is fine, and must not be stripped

- **Credential key names** — `gitlab_api_read_token`, `gitlab_api_token`,
  `gitlab_package_registry_url`, `ammonit_or_password`,
  `windcube_insights_password`, `EWS_CREDENTIALS`, `EWS_CREDENTIALS_KEYS`.
  These are the action's public interface: a consumer cannot use it without
  them. Names yes, values never.
- **The `EWS-Consulting-Private` links that are already here.** The docs point
  at `ews-github-utils` (the tool that provisions the secret) and at
  `ews-jupyter` (the reference consumer). Those predate this rule and they earn
  their place — `ews-jupyter` is the only complete worked example the docs
  have. **Do not add new ones**, and do not silently remove the existing ones
  either; that would delete the only worked example. If they should go, that is
  Fabien's call, not a tidy-up.
- Generic tooling names: `uv`, `ruff`, `pytest`, `nox`, `twine`, GitLab and
  GitHub as products.

## Before you write

Ask of every new line: **would this tell a stranger something about EWS's
internal infrastructure that they could not already see?** If yes, it does not
go here. It goes in the private hub.

One check that catches the mechanical half, before committing — absolute
paths, UNC shares, intranet hostnames, mount points and long numeric ids:

```bash
git diff --cached | grep -niE '\b[A-Za-z]:\\|\\\\[a-z0-9-]+\\|\.local\b|/mnt/[A-Za-z]\b|\b[0-9]{7,}\b'
```

Add the internal names you are actually working near as a second pass. They
are deliberately not listed in this file: a rule that enumerates the secrets it
protects publishes them itself, and this file is public too.

Neither check is exhaustive — they are a floor, not a proof. Judgment is the
actual mechanism.

## If something internal is already published

**Do not quietly delete it.** Removing the line does not remove it from
history, and a silent fix leaves nobody aware of the exposure. Say what you
found, where, and let Fabien decide. Rewriting public history is his call.

This has already happened once: an estate log issue was opened on this
repository with the standard body, which named internal repositories and local
paths. It was rewritten to a neutral note within minutes — and the original
text remains readable in the issue's public edit history. That is why this
repository deliberately has **no estate log issue**, and why one must not be
created.
