---
layout: post
title: "The silent annotation and SSE causality"
date: 2026-05-13
type: phase-update
entry_type: note
subtype: diary
projects: [claudony]
tags: [quarkus, testing, sse, cdi, annotation]
excerpt: "Two Quarkus findings from SSE test coverage: @TestSecurity that silently does nothing without an HTTP call, and why SSE causality in Claudony can't be tested through Event.fireAsync()."
---

Two Quarkus-specific findings from closing out the SSE test coverage.

`@TestSecurity(user = "test", roles = "user")` does nothing on a `@QuarkusTest`
class that never makes an HTTP call. The annotation works by registering a security
identity for the active request context — no request, no context, and Quarkus emits
no warning. The test compiles, passes, and the annotation sits there misleading the
next reader.

It turned up on two new test classes that had inherited it from the HTTP-exercising
tests in the same package. Every surrounding class needs it; those two don't. Claude
caught it in the post-implementation review pass; we removed both and noted it in
the platform protocols so it doesn't propagate.

## The registry-hooks causality problem

The trickier finding was in the SSE push tests themselves. The first draft of
`updateStatus_triggersSSEPush` asserted `received.hasSize(2)`: subscribe, trigger
the registry mutation, expect two events. That proves count. A bug that called
`subscribe()` twice would also pass. The test succeeds while proving nothing about
whether the registry mutation actually fired the change listener.

Claude pushed back and we ended up with an `AtomicInteger` counter inside the
snapshot supplier:

```java
var callCount = new java.util.concurrent.atomic.AtomicInteger(0);

broadcaster.subscribe("case-1",
        () -> callCount.incrementAndGet() == 1 ? "data: initial\n\n" : "data: updated\n\n")
    .subscribe().with(e -> { received.add(e); latch.countDown(); });

registry.updateStatus("session-1", SessionStatus.ACTIVE);

assertThat(received.get(0)).isEqualTo("data: initial\n\n");
assertThat(received.get(1)).isEqualTo("data: updated\n\n");
```

`subscribe()` calls the supplier once synchronously for the initial snapshot.
`onLifecycleEvent()` calls it again when the mutation fires. Distinct return values
mean the second event can only be `"data: updated\n\n"` if the change listener
actually ran. Content assertion instead of count assertion — the test now proves
causality.
