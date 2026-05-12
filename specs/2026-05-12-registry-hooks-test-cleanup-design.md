# Design: Registry-Hooks Test Cleanup

**Issue:** casehubio/claudony#110  
**Date:** 2026-05-12

## Changes

### 1. `RegistryHooksStrategyIntegrationTest` — add negative test

`SessionRegistry.register()` does not call `notifyListeners()` — by design, only mutations (updateStatus, remove) trigger SSE pushes. This is unverified. Add:

```java
@Test
void register_doesNotTriggerSSEPush() throws Exception {
    var received = new CopyOnWriteArrayList<String>();

    broadcaster.subscribe("rh-case-3", () -> "data: snap\n\n")
            .subscribe().with(received::add);

    registry.register(caseSession("rh-s3", "rh-case-3"));

    Thread.sleep(200);
    assertThat(received).hasSize(1); // only the initial snapshot
}
```

Subscribe before registering so the initial snapshot is captured. Wait 200ms (same pattern as `EventsOnlyStrategyTest`). Assert exactly 1 item — no second push means `register()` did not notify.

### 2. `HybridDefaultConfigTest` — remove `@TestSecurity`

The test injects a CDI bean and calls a package-private method directly. No HTTP endpoint is involved. `@TestSecurity` has no effect. Remove:
- `import io.quarkus.test.security.TestSecurity;`
- `@TestSecurity(user = "test", roles = "user")`
