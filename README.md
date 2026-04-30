# rbw-proxy

Credential proxy for AI coding agents sandboxed behind Seatbelt / bubblewrap.
File-based IPC, per-project secret manifests, audit logging.

## Status

Pre-alpha. Scaffolding only — daemon, client, and manifest loader land in
follow-up PRs (see Linear G-561).

## Why

Claude Code's Seatbelt sandbox blocks Unix domain socket IPC
([anthropics/claude-code#52471](https://github.com/anthropics/claude-code/issues/52471)),
which breaks [`rbw`](https://github.com/doy/rbw) — it talks to its agent over
a Unix socket. Today's workarounds (env-var pre-caching, disabling the sandbox)
all leak: any plaintext secret an agent can read is a secret that prompt
injection can exfiltrate.

`rbw-proxy` sits **outside** the sandbox and serves secrets via file-based IPC
the sandbox **does** allow. Agents request a named secret; the proxy validates
against a per-project manifest, fetches via `rbw`, writes a short-lived
response file, and audits every access.

## Architecture

```
┌─────────────────────────────────────────┐
│  Agent Sandbox (Seatbelt)               │
│  Writes:  $TMPDIR/requests/{uuid}.req   │
│  Polls:   $TMPDIR/responses/{uuid}.res  │
└─────────────────────────────────────────┘
           │ file-based IPC
           ▼
┌─────────────────────────────────────────┐
│  rbw-proxy daemon (OUTSIDE sandbox)     │
│  1. Watch requests/                     │
│  2. Validate against manifest           │
│  3. Call rbw get --field <field>        │
│  4. Write response (30s TTL)            │
│  5. Append to JSONL audit log           │
│  6. Clean up after TTL                  │
└─────────────────────────────────────────┘
```

## Layout

```
src/        daemon + library code
bin/        executables (daemon entry, client shell script)
tests/      node --test suites
manifests/  example secrets manifests
```

## License

MIT
