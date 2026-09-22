# Agent Pool Management — Design Spec

**Issue:** casehubio/claudony#205
**Date:** 2026-09-21
**Status:** Draft

---

## Problem

Workers in the CaseHub ecosystem need LLM backing — either API connections (Anthropic, OpenAI, Gemini, etc.) or CLI agent sessions (Claude Code in tmux). Today both paths are 1:1: each worker creates a fresh LLM instance. There is no pooling, no scaling, no unified routing, and no cost or utilisation tracking.

The platform already has a comprehensive agent infrastructure in `casehub-platform`:

- `AgentProvider` / `AgentBackend` SPIs (`agent-api`) — the caller-facing and backend-facing interfaces
- `RoutingAgentProvider` (`agent-router`) — dispatches to `AgentBackend` implementations by the `model` key on `AgentSessionConfig`
- `agent-gate` — CDI `@Decorator` wrapping `AgentProvider` with token bucket + concurrency gate rate limiting, session leak detection
- Seven backend implementations: `agent-claude`, `agent-openai`, `agent-codex`, `agent-gemini`, `agent-gemini-cli`, `agent-langchain4j`
- `ModelRegistry` (`platform-api`) — model catalog with descriptors and queries

**None of these are currently deployed to claudony.** Only `NoOpAgentProvider @DefaultBean` is active (logs a warning to add `agent-router` + backends). Meanwhile, the engine's `AgentConverter` bypasses the platform stack entirely by constructing LangChain4j `ChatModelProvider` instances inline via a 5-way switch on provider name.

The platform's agent infrastructure handles routing, rate limiting, and backend dispatch — but it lacks pool lifecycle semantics: pre-warming, idle eviction, health monitoring, and capacity-bounded resource management.

## Goal

1. Deploy the existing platform agent stack to claudony (routing, gating, backends)
2. Add agent pool lifecycle management for CLI sessions — pre-warming, capacity bounds, health monitoring, observability
3. Converge the engine's `AgentConverter` onto the platform agent stack via existing LangChain4j interop

---

## Architecture

### Existing Platform Agent Stack (to be deployed to claudony)

```
Caller
  → AgentProvider                    (SPI — agent-api)
    → agent-gate @Decorator          (concurrency gate, rate limiting — agent-gate)
      → RoutingAgentProvider         (dispatches by model key — agent-router)
        → AgentBackend by key()      (CDI Instance<AgentBackend>)
          → agent-claude             (Claude CLI via claude-code-sdk)
          → agent-openai             (native OpenAI SDK)
          → agent-gemini             (native Google GenAI SDK)
          → agent-langchain4j        (catch-all fallback, bidirectional interop)
          → ClaudonyAgentBackend     (NEW — tmux CLI sessions with pool lifecycle)
```

Currently only `NoOpAgentProvider @DefaultBean` is active on claudony. Deploying `agent-router` + at least one backend displaces it automatically via CDI.

### Agent Pool Layer (internal to ClaudonyAgentBackend)

```
ClaudonyAgentBackend implements AgentBackend
  → AgentPool                        (internal — capacity-bounded tmux session pool)
    → TmuxService                    (existing — session create/destroy/health)
    → SessionRegistry                (existing — session tracking)
```

Pool lifecycle (pre-warming, idle eviction, health checks) is an implementation detail of `ClaudonyAgentBackend`. Callers interact only with `AgentProvider.invoke()` / `openSession()` — pool semantics are fully transparent.

### Repos and Responsibilities

| Repo | What changes |
|------|-------------|
| **claudony** | Deploy `agent-router`, `agent-gate`, `agent-claude`, `agent-langchain4j` (pom.xml). New `ClaudonyAgentBackend` with internal `AgentPool`. Observability endpoints. |
| **engine** | `AgentConverter` convergence onto `AgentProvider` via `agent-langchain4j` interop (replaces inline `ChatModelProvider` construction) |

No new platform SPIs are introduced. Pool lifecycle is claudony-internal.

---

## Component Design

### 1. Deploy Platform Agent Stack (claudony — pom.xml)

Add compile dependencies to claudony:

| Artifact | What it activates |
|----------|-------------------|
| `casehub-platform-agent-router` | `RoutingAgentProvider` — dispatches to `AgentBackend` by `model` key; displaces `NoOpAgentProvider` |
| `casehub-platform-agent-gate` | CDI `@Decorator` — token bucket + concurrency gate rate limiting; session leak detection via `@Scheduled` reaper |
| `casehub-platform-agent-claude` | `AgentBackend` "claude" — Claude CLI subprocess via `claude-code-sdk` |
| `casehub-platform-agent-langchain4j` | `AgentBackend` "langchain4j" — catch-all fallback; bidirectional LangChain4j interop (`AgentProviderChatModel` wraps `AgentProvider` as LangChain4j `ChatModel`) |

Additional backends (`agent-openai`, `agent-gemini`, `agent-codex`, `agent-gemini-cli`) can be added by classpath presence as needed — no code changes required.

### 2. ClaudonyAgentBackend (claudony)

Registers Claudony CLI tmux sessions as a first-class `AgentBackend`, making them routable through `RoutingAgentProvider` alongside API backends.

```java
@ApplicationScoped
public class ClaudonyAgentBackend implements AgentBackend {

    @Override
    public String key() { return "claudony"; }

    @Override
    public Multi<AgentEvent> invoke(AgentSessionConfig config) {
        // Acquire pre-warmed tmux session from internal pool
        // Execute one-shot command, stream AgentEvent results, release session
    }

    @Override
    public AgentSession openSession(AgentSessionInit init) {
        // Acquire pre-warmed tmux session from internal pool
        // Return interactive session handle
        // Session.close() triggers pool release
    }
}
```

The returned `AgentSession` wraps pool release into its `close()` method — when callers close the session (via try-with-resources or explicit `close()`), the underlying tmux session is released back to the pool. No separate acquire/release API is exposed.

### 3. AgentPool (claudony — internal to ClaudonyAgentBackend)

Capacity-bounded pool of pre-warmed tmux sessions. Not exposed as a platform SPI — an implementation detail of `ClaudonyAgentBackend`.

**Configuration** via Quarkus `@ConfigMapping` (prefix `claudony.agent-pool`):

| Property | Type | Default | Description |
|----------|------|---------|-------------|
| `min` | int | 0 | Pre-warm count at startup |
| `max` | int | 10 | Capacity ceiling |
| `health-check-interval` | Duration | 10s | Health check frequency |
| `idle-timeout` | Duration | 15m | Destroy idle sessions after this |
| `acquire-timeout` | Duration | 10s | Max wait for available session |
| `shutdown-drain-timeout` | Duration | 30s | Max wait for in-flight sessions on shutdown |

**Lifecycle behaviours:**

- **Pre-warming:** On startup (`@Observes StartupEvent`), creates `min` tmux sessions. Pre-warming happens after `TmuxService` is available (guaranteed by CDI wiring — `TmuxService` has no async dependencies). If pre-warming fails (e.g., tmux not installed), the backend starts **degraded**: log warning, pool health `DEGRADED`, sessions created on-demand. Server does not fail to start.
- **On-demand creation:** When all idle sessions are exhausted, creates new ones up to `max`. Blocks (up to `acquire-timeout`) if at max capacity. On timeout, throws `AgentSessionLimitException` (existing platform type) — callers decide whether to retry, fall back, or fail fast.
- **Idle eviction:** `@Scheduled` task destroys sessions idle longer than `idle-timeout`, down to `min`.
- **Session recycling:** When a CLI session is released, the pool does NOT reuse it as-is — tmux sessions carry state (working dir, env, history). Released sessions are destroyed and replaced with fresh pre-warmed sessions up to `min`. This means the pool behaves as a "warm start" pool (pre-created sessions ready for first use) rather than a connection pool (reused across requests).
- **Graceful shutdown:** `@Observes ShutdownEvent` drains active sessions — waits up to `shutdown-drain-timeout` for in-flight work to complete, then destroys all pooled sessions.

### 4. Health Check Protocol

Health checks are `@Scheduled` at the pool's `health-check-interval`.

| Check | Method | Cost |
|-------|--------|------|
| tmux session alive | `tmux has-session -t <name>` | Free — local process check |
| session responsive | last-active timestamp within 2× health-check-interval | Free — in-memory check |

**No API calls for health checks.** API backend health (for `agent-claude`, `agent-openai`, etc.) is the responsibility of those platform backends, not this pool. The pool only manages tmux CLI session health.

**Circuit breaker:** If 3 consecutive health checks fail for a session, the session is evicted and replaced. If all sessions in the pool fail health checks simultaneously:
1. Pool status transitions to `UNHEALTHY`
2. CDI event fired for dashboard alerting
3. Replacement attempts use exponential backoff: 30s → 1m → 2m → 5m (same pattern as `PeerRegistry` in the existing fleet layer)
4. Pool does NOT attempt to create `max` replacements simultaneously — creates one at a time with backoff

### 5. Engine Convergence (engine)

`AgentConverter.toChatModelProviderFromNode()` currently constructs LangChain4j `ChatModelProvider` instances inline via a switch on provider type name (`openai`, `anthropic`, `ollama`, `mistralai`/`mistral`, `googleaigemini`/`gemini`). This bypasses `RoutingAgentProvider` and therefore bypasses pool management, rate limiting, and observability.

**Convergence path** (leverages existing `agent-langchain4j` interop):

1. Deploy `agent-router` + relevant backends to the engine's classpath
2. `AgentConverter` resolves models through `AgentProvider` instead of constructing them directly:

```java
// Before (inline construction — 5-way switch):
ChatModelProvider modelProvider = toChatModelProviderFromNode(providerConfigNode, providerType);

// After (routed through platform agent stack):
// AgentProviderChatModel (from agent-langchain4j) wraps AgentProvider as ChatModel
ChatModel chatModel = agentProviderChatModel;
```

3. The `AgentProviderChatModel` from `agent-langchain4j` bridges `AgentProvider` to LangChain4j `ChatModel` — the existing interop layer handles event conversion via `AgentEventBridge`
4. YAML-defined agents automatically get pool management and rate limiting when their model resolution goes through `RoutingAgentProvider`

### 6. Observability (claudony)

Pool status exposed via REST:

```
GET /api/agent-pools              → all pool statuses
GET /api/agent-pools/{backendKey} → single pool status + instance details
```

`PoolStatus` response:

```java
public record PoolStatus(
    String backendKey,
    int min,
    int max,
    int active,
    int idle,
    int total,
    PoolHealth health  // HEALTHY, DEGRADED, UNHEALTHY
) {}
```

Metrics tracked per pool:
- `agent.pool.active` — gauge: currently acquired sessions
- `agent.pool.idle` — gauge: available sessions
- `agent.pool.acquire.count` — counter: total acquisitions
- `agent.pool.acquire.wait` — histogram: time waiting for a session
- `agent.pool.health` — gauge: 0 (unhealthy) to 1 (healthy)
- `agent.pool.eviction.count` — counter: sessions evicted (health or idle)

### 7. Integration with WorkerProvisioner

`ClaudonyWorkerProvisioner` currently creates tmux sessions directly via `TmuxService`. With agent pool management:

1. `ClaudonyWorkerProvisioner` injects `AgentProvider` (the platform SPI) instead of calling `TmuxService` directly
2. For worker provisioning, it calls `agentProvider.openSession(init)` which routes through `RoutingAgentProvider` → `ClaudonyAgentBackend` → pool
3. The pool returns a pre-warmed or freshly created session
4. On worker completion, `session.close()` triggers pool release automatically (via the `AgentSession.close()` contract)
5. The pool decides whether to destroy and replace the session (CLI sessions carry state) or keep it

This change also means workers can be backed by any `AgentBackend` — not just tmux CLI sessions. A worker could be backed by `agent-claude` (subprocess via claude-code-sdk) or `agent-openai` (API) if configured to resolve to those backends.

---

## Sequencing (implementation order)

1. **Claudony: Deploy platform agent stack** — add `agent-router`, `agent-gate`, `agent-claude`, `agent-langchain4j` to pom.xml. Verify `NoOpAgentProvider` is displaced. Lowest risk — purely additive.
2. **Claudony: ClaudonyAgentBackend + AgentPool** — register CLI sessions as a backend with internal pool lifecycle. Test independently.
3. **Claudony: Wire WorkerProvisioner** — migrate `ClaudonyWorkerProvisioner` from direct `TmuxService` calls to `AgentProvider`. Test provisioning flow end-to-end.
4. **Claudony: Observability** — REST endpoints, metrics.
5. **Engine: AgentConverter convergence** — last, highest risk. Migrate inline `ChatModelProvider` construction to `AgentProviderChatModel`. Test each provider.

Step 5 (engine convergence) is sequenced last because it changes how all YAML-defined agents resolve models — widest blast radius. Steps 1-4 can be tested and validated independently.

---

## What This Does NOT Cover

Each deferred item is filed as a GitHub issue to ensure tracking:

- **Auto-scaling policies** (target-tracking, step scaling) — the min/max model is sufficient initially. casehubio/claudony#TBD
- **Qhorus mesh routing integration** — fleet-managed LLM instances as Qhorus participants. casehubio/claudony#TBD
- **Ops perspective integration** (scaffold#52) — fleet provisioning in management UI. casehubio/claudony#TBD
- **Eidos agent identity integration** — org structures defining which agents exist, pool providing LLM backing. casehubio/claudony#TBD
- **Cross-machine pool distribution** — pools are local to a Claudony instance. `PeerRegistry` handles multi-machine fleet at the Claudony level; pool distribution across machines is a separate concern. casehubio/claudony#TBD
- **Cost budgeting/limits** — cost tracking (tokens consumed) is not included in this phase. casehubio/claudony#TBD
- **Model fallback chains** — "if opus is unavailable, fall back to sonnet." This is a routing concern, not a pool concern. Can be added to `RoutingAgentProvider` independently. casehubio/claudony#TBD

---

## References

- `AgentProvider`, `AgentBackend`, `AgentEvent`, `AgentSession`, `AgentSessionConfig`, `AgentSessionInit`, `AgentSessionLimitException` (`casehub-platform-agent-api`) — existing agent SPIs and types
- `RoutingAgentProvider` (`casehub-platform-agent-router`) — existing routing/resolution by model key
- `agent-gate` (`casehub-platform-agent-gate`) — existing CDI `@Decorator` rate limiter
- `agent-langchain4j` (`casehub-platform-agent-langchain4j`) — bidirectional LangChain4j interop
- `ModelRegistry`, `ModelDescriptor`, `ModelQuery` (`casehub-platform-api`) — existing model catalog
- `NoOpAgentProvider` (`casehub-platform`) — `@DefaultBean` placeholder, displaced by `agent-router`
- `AgentConverter`, `ChatModelProvider` (`casehub-engine-api`) — inline model construction to be converged
- `ClaudonyWorkerProvisioner`, `TmuxService`, `SessionRegistry` (claudony) — existing CLI session provisioning
- `PeerRegistry` (claudony `server/fleet/`) — peer mesh fleet management (distinct from agent pool management)
- HikariCP — prior art for capacity-bounded pool patterns
