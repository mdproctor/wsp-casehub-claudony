---
layout: post
title: "One Dashboard, All Sessions"
date: 2026-04-15
type: phase-update
entry_type: note
subtype: diary
projects: [claudony]
tags: [fleet, security, architecture]
excerpt: "Fleet Phase 1 extends Claudony to manage sessions across multiple instances — and clears three deferred embarrassments: a hardcoded encryption key committed to the repo, a 30-minute session timeout, and three undocumented ADRs."
---

## The fleet was inevitable

When I started Claudony, I was solving a specific problem: keep Claude Code sessions alive when the browser closes, and get access to them from any device. One machine, one server, one dashboard. Simple.

But I always knew where it was going. The ecosystem design — CaseHub coordinating workers, Qhorus carrying messages, Claudony as the integration layer — assumes multiple instances running in parallel. Agents on different machines, each managing a pool of sessions, all visible from one place. The architecture was designed for it. The code wasn't there yet.

Today it is, at least for Phase 1.

## Three things before the fleet

Before the fleet work, we cleared three deferred items that were quietly embarrassing.

The session encryption key was first. The prod cookie key — the thing that keeps you logged in across server restarts — was a hardcoded string committed to the repo. Every Claudony deployment in the world was using `XrK9mP2vLq8nT5wY3jH7bN4cF6dA1eZ0`. We replaced it with an `EncryptionKeyConfigSource` — a MicroProfile `ConfigSource` that runs before CDI, generates a 256-bit random key on first boot, persists it to `~/.claudony/encryption-key` with `rw-------` permissions, and loads it on subsequent starts. No configuration required. Each deployment gets its own key.

Session timeout came second. The WebAuthn default is 30 minutes of inactivity. For a terminal tool you keep open all day, that means re-authenticating via passkey constantly. One property: `quarkus.webauthn.session-timeout=${claudony.session-timeout}`, default `P7D`. Done.

Then three pre-1.0 ADRs — terminal streaming strategy, MCP transport choice, authentication mechanism — documenting decisions made months ago that were never formally recorded.

## The fleet design

The core idea: every Claudony instance is both a peer server and a peer client. No master, no hub. A symmetric mesh where any node can see any other node's sessions.

Three discovery mechanisms feed the same `PeerRegistry`:

- **Static config** — `claudony.peers=http://mac-mini:7777`. Perfect for Docker Compose, where service names are stable.
- **Manual registration** — `POST /api/peers`. Add one seed and the registry exchanges peer lists on first contact; the mesh fills in.
- **mDNS** — `_claudony._tcp.local.` on LAN. Disabled by default; scaffolded but not fully wired yet.

Each peer gets a circuit breaker: three consecutive failures open the circuit, exponential backoff from 30s to 5 minutes, recovery on successful health check. Stale sessions from an unreachable peer are served from cache with `stale: true`. The dashboard stays usable; you see old data instead of an error.

Session federation is the keystone. `GET /api/sessions` fans out to all healthy peers in parallel with a 2-second timeout per peer. A `?local=true` parameter stops recursive federation — when peer A calls peer B, B only returns its own sessions:

```java
@GET
public List<SessionResponse> list(@QueryParam("local") @DefaultValue("false") boolean localOnly) {
    var result = new ArrayList<>(registry.all().stream()
            .map(s -> SessionResponse.from(s, config.port()))
            .toList());
    if (localOnly) return result;
    // fan out to healthy peers...
}
```

## What the subagent reviews caught

We built this with subagent-driven development — twelve tasks, one subagent per task, two-stage review after each. Three things surfaced that I wouldn't have caught in a self-review pass.

The `PATCH /api/peers/{id}` endpoint was returning `Optional<PeerRecord>` directly in the response body. Jackson serialises `Optional` as `{"present":true,"empty":false}`, not as the record. The test happened to pass because of Jackson's module configuration in that context. Wrong either way.

The `DELETE` endpoint had a TOCTOU window — check if peer exists, then remove it. Two separate operations. A concurrent removal between the two produces a 405 instead of a 404. Fixed to a single `.map()` chain.

The most interesting: `@RegisterProvider(FleetKeyClientFilter.class)` on the `PeerClient` interface is not guaranteed to be honoured by programmatic `RestClientBuilder.newBuilder()` calls. The MicroProfile spec covers injected clients; programmatic builders are implementation-defined. If the filter silently doesn't run, the fleet key header is missing, every peer call returns 401, and the circuit breakers open. The fix is explicit `.register(FleetKeyClientFilter.class)` on each builder call. Three sites needed it. That one went into the garden.

## 207 tests, a Dockerfile, and a two-node compose

The fleet adds 45 tests — unit and QuarkusTest — covering the circuit breaker state machine, peer deduplication by URL, atomic `peers.json` persistence, REST CRUD, session federation with stale cache fallback, and fleet key auth.

The Dockerfile targets JVM mode: `eclipse-temurin:21-jre-alpine` plus `tmux`. The compose example wires two nodes with static peer config and named volumes for `~/.claudony/`. Running `export CLAUDONY_FLEET_KEY=$(openssl rand -base64 32) && docker compose up` produces a working two-node fleet.

Phase 2 — dashboard fleet panel, session instance badges, PROXY WebSocket bridge for peers behind NAT — is next.
