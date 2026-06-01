# Oversight Channel — Full Type Map
**Issue:** claudony#142  
**Date:** 2026-06-01

---

## MessageType semantics (from Qhorus `MessageType.java`)

| Type | Commitment effect | Agent visible? | Notes |
|------|------------------|----------------|-------|
| `COMMAND` | Opens commitment (OPEN) | Yes | Ask for action; receiver must DONE or FAILURE |
| `QUERY` | Opens commitment (OPEN) | Yes | Ask for information; receiver must RESPONSE or DECLINE |
| `RESPONSE` | Fulfills commitment (FULFILLED) | Yes | Answers a QUERY |
| `DONE` | Fulfills commitment (FULFILLED) | Yes | Signals successful completion of a COMMAND |
| `DECLINE` | Declines commitment (DECLINED) | Yes | Refuses a QUERY or COMMAND |
| `FAILURE` | Fails commitment (FAILED) | Yes | Signals unsuccessful termination of a COMMAND |
| `STATUS` | Acknowledges commitment (extends deadline) | Yes | Progress report; does not close the commitment |
| `HANDOFF` | Delegates commitment (DELEGATED) | Yes | Transfers obligation to named target |
| `EVENT` | No commitment effect | **No** | Observer-only telemetry; excluded from agent context by default |

---

## Oversight patterns — type requirements

### Pattern A: Agent asks human for input (QUERY → human)
| Step | Actor | Type | Commitment state |
|------|-------|------|-----------------|
| Agent requests decision | Agent | QUERY | OPEN |
| Human asks clarifying question | Human | QUERY | OPEN (new nested commitment) |
| Agent clarifies | Agent | RESPONSE | FULFILLED (nested) |
| Human takes time to review | Human | STATUS | deadline extended |
| Human decides | Human | RESPONSE | FULFILLED |
| — or — | Human | DECLINE | DECLINED |
| Human can't decide, delegates | Human | HANDOFF | DELEGATED |

Types required: `QUERY, RESPONSE, DECLINE, STATUS, HANDOFF`

---

### Pattern B: Engine gate — ActionRiskClassifier (engine#402)
| Step | Actor | Type | Commitment state |
|------|-------|------|-----------------|
| Engine pauses workflow, agent posts approval request | Agent | COMMAND | OPEN |
| Human asks for context before deciding | Human | QUERY | OPEN (nested) |
| Agent provides context | Agent | RESPONSE | FULFILLED (nested) |
| Human takes time | Human | STATUS | deadline extended |
| Human approves | Human | RESPONSE | FULFILLED |
| — or — | Human | DECLINE | DECLINED |
| Human delegates to another reviewer | Human | HANDOFF | DELEGATED |

Types required: `COMMAND, RESPONSE, DECLINE, STATUS, QUERY, HANDOFF`

---

### Pattern C: Human injects directive
| Step | Actor | Type | Commitment state |
|------|-------|------|-----------------|
| Human issues instruction to agent | Human | COMMAND | OPEN |
| Agent progresses, reports status | Agent | STATUS | deadline extended |
| Agent completes | Agent | DONE | FULFILLED |
| — or agent fails — | Agent | FAILURE | FAILED |

Types required: `COMMAND, DONE, FAILURE, STATUS`

---

### Pattern D: Human delegates to another human (continuation of any pattern)
| Step | Actor | Type | Commitment state |
|------|-------|------|-----------------|
| Human can't decide | Human | HANDOFF | DELEGATED |
| New human decides | Human | RESPONSE / DECLINE | FULFILLED / DECLINED |

Types required: `HANDOFF, RESPONSE, DECLINE`

---

## Full required set

Union of all patterns:

```
COMMAND, QUERY, RESPONSE, DONE, DECLINE, FAILURE, STATUS, HANDOFF
```

**Not required: `EVENT`**

### Why EVENT does not belong on oversight

1. `EVENT` is `NOT delivered to agent context` — governance participants (agent + human) cannot see it.
2. `MessageService.pollAfter()` excludes `EVENT` by default — standard check_messages calls never return it.
3. Watchdog deadline alerts have a configurable `notificationChannel` — these should route to `observe`, not `oversight`. Routing them to `oversight` is a convention error: the EVENT is invisible to oversight participants anyway.
4. The design principle "oversight is a governance space, not a progress log" maps precisely to "no telemetry on the governance channel."

---

## Channel invariants — symmetric model

| Channel | Invariant | Expression |
|---------|-----------|-----------|
| `observe` | No obligations created | `allowedTypes = EVENT` |
| `oversight` | No pure telemetry | `allowedTypes = COMMAND,QUERY,RESPONSE,DONE,DECLINE,FAILURE,STATUS,HANDOFF` |
| `work` | No constraint | `allowedTypes = null` |

The observe and oversight constraints are **reciprocal**:
- Observe says: *this channel must never create an obligation*
- Oversight says: *this channel must never contain pure telemetry*

`allowedTypes` is an architectural invariant, not a categorisation label. Only set it when there is a hard constraint that must never be violated. Work has no such constraint — it is the open coordination space.

---

## Implementation options

### Option 1 — Enumerate 8 types in allowedTypes (no Qhorus change)
```java
new ChannelSpec("oversight", ChannelSemantic.APPEND,
    Set.of(COMMAND, QUERY, RESPONSE, DONE, DECLINE, FAILURE, STATUS, HANDOFF),
    "Human governance — all obligation-carrying types; no telemetry")
```
Correct but fragile: adding a new non-EVENT type to `MessageType` requires updating this set manually.

### Option 2 — Add `deniedTypes` to Qhorus (small schema change)
```java
// ChannelSpec gains a deniedTypes field:
new ChannelSpec("oversight", ChannelSemantic.APPEND,
    null,                      // allowedTypes: open
    Set.of(EVENT),             // deniedTypes: no telemetry
    "Human governance — all obligation-carrying types; no telemetry")
```
Symmetric with `allowedTypes`. Oversight is `deniedTypes = EVENT`. New obligation-carrying types added to the enum are automatically included. Requires: new `denied_types TEXT` column in Qhorus `channel` table, updated `StoredMessageTypePolicy`.

### Option 3 — Channel roles in Qhorus (bigger change)
Introduce a `channelRole` enum: `COORDINATION` (null) / `GOVERNANCE` (all except EVENT) / `TELEMETRY` (EVENT only).  
Cleanest API; role conveys intent; no type lists needed. Larger Qhorus change.

---

## Current bugs (before fix)

1. **RESPONSE/DECLINE blocked on oversight** — engine#402 gate cannot close commitments.
2. **STATUS blocked** — humans cannot extend deadlines on oversight commitments.
3. **DONE/FAILURE blocked** — agents cannot discharge COMMANDs issued by humans.
4. **HANDOFF blocked** — humans cannot delegate governance decisions.
5. **EVENT blocked** — Watchdog deadline EVENTs blocked (though these should route to `observe` anyway — convention fix, not just a type fix).
6. **Spec inconsistency** — mesh framework spec line 708 says `allowedTypes = "QUERY,COMMAND"` but line 300 shows RESPONSE on oversight.
