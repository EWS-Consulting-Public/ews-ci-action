# CLAUDE.md — ews-ci-action

@AGENTS.md

Claude Code entrypoint. Shared project instructions live in
[`AGENTS.md`](AGENTS.md) (imported above).

**This repository is public.** Everything you write here — files, commit
messages, issue text — is world-readable and permanent. Read
`.cursor/rules/public-repo-boundary.mdc` before writing.

## Claude-local pointers

- **`.claude/` is generated — do not edit it.** Author under
  [`.cursor/`](.cursor/), then
  `uv run --no-project python scripts/sync_agent_config.py`. `prek` runs
  `--check`. `--no-project` because there is no `pyproject.toml`.
- Rules: [`.claude/rules/`](.claude/rules/) — generated from
  `.cursor/rules/*.mdc`. Claude loads **all** of them, so each carries an
  **Applies to** header naming the paths it is scoped to in Cursor. Skip a rule
  whose paths your change does not touch.
- Policy: [`.claude/rules/agent-config-sync.md`](.claude/rules/agent-config-sync.md)
- Memory store, if one exists on this host, is per-repo and per-host and is
  **not** part of this repository. Nothing durable belongs there — and nothing
  internal belongs here.
