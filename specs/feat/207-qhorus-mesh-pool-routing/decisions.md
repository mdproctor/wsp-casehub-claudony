## D1: Pool→Mesh bridge mechanism

**Choice:** CDI event observer — pool fires CDI events on lifecycle transitions, an observer in `claudony-app` calls Qhorus `InstanceService`
**Alternatives:**
- Direct injection — AgentSessionManager calls InstanceService directly. Couples pool to Qhorus; wrong module boundary.
- Polling reconciliation — scheduled job reconciles. Delayed visibility, more complex.
**Rationale:** CDI events are the established Claudony pattern for cross-concern bridging (WorkerCaseLifecycleEvent, CaseEventBroadcaster). Keeps pool module clean of Qhorus dependency. Observer is independently testable.
**Trade-offs:** Slightly more indirection than direct injection; events could theoretically be missed (mitigated by CDI guarantees within a single JVM).
**Sources:** AgentSessionManager.java, InstanceService.java, WorkerCaseLifecycleEvent.java pattern
**Exploration:** quick
**Status:** captured

## D2: Event source mechanism

**Choice:** Callback listener interface (`SessionLifecycleListener`) in `claudony-casehub`, with a CDI implementation in `claudony-app` that fires CDI events
**Alternatives:**
- Wrapper/decorator — CDI wrapper around AgentSessionManager. Doubles API surface.
- Make AgentSessionManager CDI-aware — inject Event<> directly. Breaks POJO nature and testing simplicity.
**Rationale:** Follows the established SPI callback pattern (WorkerStatusListener, SessionOperations). AgentSessionManager stays a testable POJO. The listener is optional — pools work fine without mesh integration.
**Trade-offs:** One extra interface. Acceptable given it's the standard pattern.
**Sources:** WorkerStatusListener SPI, SessionOperations SPI
**Exploration:** quick
**Depends on:** D1 (CDI event observer)
**Status:** captured

## D3: Pool-to-Qhorus capability mapping

**Choice:** Capability tag `pool:{poolName}` (e.g. `pool:code-reviewer`). Distinguishes pool-managed instances from self-registered ones. Instance description includes pool name + session identity. `claudonySessionId` set to session's `instanceId`.
**Alternatives:**
- Agent name only — simpler but can't distinguish pool from self-registered instances.
- Both pool and agent tags — more discoverable but adds unnecessary complexity.
**Rationale:** The `pool:` prefix makes mesh routing queries unambiguous: `findByCapability("pool:code-reviewer")` returns only pool-managed instances. Self-registered instances use unprefixed capabilities.
**Trade-offs:** Callers must know the prefix convention. Acceptable — it's a namespace, not a leaky abstraction.
**Sources:** InstanceService.findByCapability(), AgentPoolDefinition.AgentConfig
**Exploration:** quick
**Depends on:** D1 (CDI event observer)
**Status:** captured
