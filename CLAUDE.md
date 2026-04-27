# rbw-proxy — project memory for Claude Code

## What this is

Credential proxy that bridges sandboxed AI agents and `rbw` (Bitwarden CLI)
via file-based IPC. Sits outside the sandbox; the agent never sees a plaintext
secret in its environment.

Tracking ticket: **Linear G-561** (team `G`, project "Mac Setup & Environment").

## Repo conventions

- Branch naming: `G-561/<short-description>` (or `G-XXX/...` for sub-tickets)
- Every commit prefixed with the Linear ticket id (e.g. `G-561: ...`)
- Never commit directly to `main` — always via PR
- License: MIT
- Node.js >=20, CommonJS (`"type": "commonjs"`), zero runtime deps
- Tests use `node --test` (built-in runner — no jest/mocha)

## Architecture invariants

- Daemon runs **outside** the sandbox; client runs inside
- IPC is files in `$TMPDIR/rbw-proxy/{requests,responses}/<uuid>.{req,res}`
- Responses are deleted by the daemon after a hard TTL (default 30s)
- Every request → one append to the JSONL audit log
- Manifest is the only source of truth for which secrets a project may read

## What lives where

- `src/daemon.js` — directory watcher, request handler, cleanup loop
- `src/manifest.js` — manifest loader + validator
- `src/audit.js`    — JSONL audit log writer
- `bin/rbw-proxy`   — daemon entry point
- `bin/rbw-proxy-client.sh` — shell client agents call
- `tests/`          — one `*.test.js` per module

## Security non-negotiables

- Never log secret **values** — only names, fields, project, pid, timestamp
- Reject manifest entries that aren't on the project's allowlist
- Rate-limit per project (configurable)
- No `child_process.exec` with interpolated input — always use `execFile`
