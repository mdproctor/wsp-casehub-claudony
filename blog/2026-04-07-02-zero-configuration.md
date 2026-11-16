# RemoteCC — Zero Configuration

**Date:** 2026-04-07
**Type:** phase-update

---

## What I was trying to achieve: close the last deployment blocker

The previous entry ended on one open item: how does the agent API key get provisioned
on a fresh Mac Mini? Without it, server and agent can't authenticate with each other
and nothing works.

I wanted zero manual setup for the common case — same machine, development mode.
That meant auto-generation.

## What I believed going in: the design was clear, the implementation would be routine

The resolution order came together quickly in design: check the config property first
(explicit always wins), then check `~/.remotecc/api-key` (same-machine auto-discovery),
then generate if absent. Server generates, agent reads or warns. New `ApiKeyService`
bean, injected wherever the key is needed.

The constraint that made this interesting: Quarkus config resolution is immutable
after startup. You can't inject a runtime-generated key back into the config system.
`ApiKeyService` has to sit outside it and serve the resolved key at request time.

## The timing trap buried in the auth wiring

We built `ApiKeyService` as a pure unit test first — `@TempDir`, mocked
`RemoteCCConfig`, same pattern as `CredentialStore`. Eight tests covering
the resolution branches, file permissions (600), blank-file handling, config
precedence. A code quality review caught one real issue before commit: `persistKey()`
was void and always set `resolvedKey`, so a failed write would still trigger the
first-run banner claiming the key was saved. Fixed to return `boolean`; the banner
only fires if persistence succeeded.

The more interesting catch came when wiring `ApiKeyService` into `ApiKeyAuthMechanism`.
The implementer had added a `@PostConstruct autoInit()` beyond what the task specified.
The reason: in `@QuarkusTest`, `HttpAuthenticationMechanism` implementations process
requests before `@Observes StartupEvent` fires.

`ApiKeyAuthMechanism` calls `apiKeyService.getKey()` at request time. `initServer()` —
which sets `resolvedKey` — is called from `ServerStartup.onStart()`, a `StartupEvent`
observer. First test request arrives, `resolvedKey` is still empty, every authenticated
call returns 401.

The fix: `@PostConstruct` that loads the config key at CDI construction time, before
any observers fire. No file I/O — just `loadFromConfig()`. When `initServer()` runs
later, it re-evaluates from the top, finds the already-set key, returns early.
Idempotent. Nothing in the Quarkus docs warns about this ordering.

## First boot

On first server run with no key configured:

```
================================================================
REMOTECC — API Key Generated (first run)
  Key:      remotecc-550e8400e29b41d4a716446655440000
  Saved to: /Users/you/.remotecc/api-key

  Same machine (agent + server co-located): no action needed.
  Different machine: configure the agent with —
    export REMOTECC_AGENT_API_KEY=remotecc-550e8400...
================================================================
```

Agent on the same machine finds the file at startup and loads silently. Agent on a
different machine starts degraded with a matching warning and the exact env var to set.

124 tests passing. The Mac Mini gets the binary, launchd starts it, and the key
provisions itself on first boot.
