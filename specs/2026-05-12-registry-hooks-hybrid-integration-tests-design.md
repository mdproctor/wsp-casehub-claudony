# Design: Integration Tests for Registry-Hooks and Hybrid-Default SSE Strategies

**Issue:** casehubio/claudony#108  
**Date:** 2026-05-12

---

## Context

`CaseEventBroadcaster` selects its SSE push strategy at `@PostConstruct` from the
`claudony.case-worker-update` config property. The test `application.properties` forces
`%test.claudony.case-worker-update=events-only` to avoid heartbeat noise in the default
test suite. Two paths have zero integration coverage:

1. **`registry-hooks`** — `CaseEventBroadcaster.init()` calls
   `registry.addChangeListener(this::emit)` when this strategy is active. No test verifies
   that `SessionRegistry.updateStatus()` and `remove()` actually trigger SSE pushes through
   this wiring.

2. **`hybrid` (production default)** — The `default` branch of the strategy switch produces
   `HybridStrategy`. No test verifies that the broadcaster selects it when configured as
   `hybrid` (the value set in production `application.properties`).

---

## Approach

Two separate `@QuarkusTest` classes, each with an inner `@QuarkusTestProfile` that overrides
`claudony.case-worker-update`. Each profile forces a Quarkus app restart, giving an isolated
application context with the correct strategy wired. This follows the established pattern in
`MeshParticipationSilentProfileTest`.

---

## Test Classes

Both go in `app/src/test/java/io/casehub/claudony/server/`.

---

### `RegistryHooksStrategyIntegrationTest`

**Profile override:** `claudony.case-worker-update=registry-hooks`

**Tests:**

| Method | What it asserts |
|--------|----------------|
| `selectedStrategy_isRegistryHooks` | `broadcaster.strategyType()` returns `"registry-hooks"` |
| `updateStatus_triggersSSEPush` | `registry.updateStatus()` on a case session emits a push to SSE subscribers |
| `remove_triggersSSEPush` | `registry.remove()` on a case session emits a push to SSE subscribers |

**Setup:** Each test registers a `Session` with a `caseId` and unique ID (`"rh-s1"`, `"rh-s2"`,
`"rh-s3"`). Corresponding unique case IDs (`"rh-case-1"`, `"rh-case-2"`, `"rh-case-3"`) prevent
cross-test subscriber interference.

**Push assertion pattern:** Subscribe to broadcaster for the caseId with a `CountDownLatch(2)`
(initial snapshot + registry-triggered push). Call the registry mutation. Assert latch completes
within 2 seconds and received list has 2 items.

**`remove()` note:** `SessionRegistry.remove()` calls `notifyListeners()` before
`sessions.remove()`, so the session's `caseId` is still resolvable when the listener fires.
No special handling needed.

**`@AfterEach` cleanup:**
```java
registry.all().stream().map(Session::id).toList().forEach(registry::remove);
```
Mirrors the pattern in `SessionResourceCaseEventsTest`. Cleanup-triggered `remove()` calls will
fire listeners (and attempt SSE pushes) but no test is asserting at that point — harmless.

---

### `HybridDefaultConfigTest`

**Profile override:** `claudony.case-worker-update=hybrid`

Overriding to `"hybrid"` exercises the `default` branch of the strategy switch, which is the
production code path (production `application.properties` sets `claudony.case-worker-update=hybrid`).

**Tests:**

| Method | What it asserts |
|--------|----------------|
| `selectedStrategy_isHybrid_whenConfiguredAsHybrid` | `broadcaster.strategyType()` returns `"hybrid"` |

No subscriptions, no cleanup — purely a config wiring assertion. The `HybridStrategy`'s
heartbeat behavior and push mechanics are already covered by `HybridStrategyTest` and
`CaseEventBroadcasterTest`.

---

## What Is Not Covered Here

These are already tested elsewhere and not repeated:

- Heartbeat tick behavior → `HybridStrategyTest`
- Push fan-out, null/unknown caseId guards → `CaseEventBroadcasterTest`
- SSE endpoint content-type and HTTP layer → `SessionResourceCaseEventsTest`
- `EventsOnlyStrategy` push mechanics → `EventsOnlyStrategyTest`

---

## Protocol Compliance

| Protocol | Compliance |
|----------|-----------|
| `quarkus-test-stateful-bean-isolation` | Unique IDs per test; `@AfterEach` removes all sessions |
| `quarkus-test-naming-convention` | Both files end in `Test.java` |
| `spi-testing-alternative-inner-classes` | No SPI doubles needed — testing wiring, not SPIs |
