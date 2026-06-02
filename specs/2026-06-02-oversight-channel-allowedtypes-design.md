# Design: Oversight Channel `allowedTypes` — `deniedTypes` + Full Commitment Lifecycle
**Issue:** claudony#142  
**Date:** 2026-06-02  
**Status:** Approved

---

## Problem

`NormativeChannelLayout` sets `oversight.allowedTypes = {QUERY, COMMAND}`. This is wrong for six concrete reasons:

1. `RESPONSE` and `DECLINE` are blocked — the engine gate (engine#402) cannot close approval commitments
2. `STATUS` is blocked — humans cannot extend deadlines on oversight commitments
3. `DONE` and `FAILURE` are blocked — agents cannot discharge COMMANDs issued by humans
4. `HANDOFF` is blocked — humans cannot delegate governance decisions
5. Watchdog deadline EVENTs are blocked (though these should route to `observe` anyway)
6. The mesh framework spec is internally inconsistent: line 708 says `"QUERY,COMMAND"` but line 300 shows `RESPONSE` being sent on oversight

The root cause: `allowedTypes` was set as a categorisation label ("this is a governance channel") rather than an architectural invariant ("this type must never appear here").

---

## Decision

### The invariant

Every channel in the three-channel layout enforces exactly one architectural constraint:

| Channel | Invariant | Expressed as |
|---------|-----------|-------------|
| `observe` | No obligations created | `allowedTypes = {EVENT}` |
| `oversight` | No pure telemetry | `deniedTypes = {EVENT}` ← new |
| `work` | No constraint | `allowedTypes = null` |

`observe` and `oversight` are reciprocal: observe says "no speech acts that create obligations"; oversight says "no telemetry that creates no obligations." `work` is the open coordination space.

### Why EVENT is excluded from oversight

- `EVENT` is "observer-only telemetry — NOT delivered to agent context" (MessageType Javadoc)
- `MessageService.pollAfter()` excludes EVENT by default — governance participants cannot see EVENTs on oversight
- Watchdog deadline alerts carry a configurable `notificationChannel`; they should route to `observe`, not `oversight`, where they are invisible to participants
- The design principle "oversight is a governance space, not a progress log" maps precisely to "no telemetry" — not "no types"

### Full required type set

The oversight channel participates in the full commitment lifecycle. Every pattern (agent asks human, engine gate, human injects directive, delegation) requires:

`COMMAND, QUERY, RESPONSE, DONE, DECLINE, FAILURE, STATUS, HANDOFF`

This is all obligation-carrying types — equivalently, all types except `EVENT`. See `design/oversight-type-map.md` for the per-pattern trace.

---

## Implementation

### 1. Qhorus — add `deniedTypes`

**Migration:** add `denied_types TEXT` column to `channel` table (nullable, null = no denial).

**`StoredMessageTypePolicy`:** check `deniedTypes` after `allowedTypes`. If a type appears in `deniedTypes`, reject it regardless of `allowedTypes`. Priority: denial always wins — a type cannot be simultaneously allowed and denied; callers should not create such a configuration, but if they do, deny wins.

**API:** `ChannelDetail` (MCP tool response) gains `deniedTypes` field alongside `allowedTypes`. `create_channel` MCP tool accepts `denied_types` parameter.

### 2. Claudony — `CaseChannelLayout.ChannelSpec`

Add `Set<MessageType> deniedTypes` to the `ChannelSpec` record:

```java
record ChannelSpec(
    String purpose,
    ChannelSemantic semantic,
    Set<MessageType> allowedTypes,   // null = open
    Set<MessageType> deniedTypes,    // null = no denial
    String description
)
```

### 3. `NormativeChannelLayout`

```java
new ChannelSpec("oversight", ChannelSemantic.APPEND,
    null,                          // allowedTypes: open to all obligation-carrying types
    Set.of(MessageType.EVENT),     // deniedTypes: no telemetry
    "Human governance — all obligation-carrying types; no telemetry")
```

`work` and `observe` gain `deniedTypes = null` (no denial) for completeness.

### 4. `ClaudonyCaseChannelProvider`

Pass `deniedTypes` through to Qhorus `create_channel` call when creating the oversight channel.

### 5. `NormativeChannelLayoutTest`

Replace the oversight allowedTypes assertion with two assertions:
- `allowedTypes()` is null
- `deniedTypes()` contains exactly `{EVENT}`

### 6. Mesh framework spec correction

Update `docs/specs/2026-04-27-claudony-agent-mesh-framework.md`:
- Line 708: change `"QUERY,COMMAND"` to `"all except EVENT (deniedTypes = EVENT)"`
- Line 68: update oversight description to reflect the full dialogic model and EventType exclusion
- Line 593: update the oversight row in the layout table

---

## Platform protocol

A new protocol captures the generalised rule:

> **`allowedTypes` is an architectural invariant, not a categorisation label.** Only set `allowedTypes` or `deniedTypes` when there is a hard constraint that must be enforced across all scenarios — not to express what a channel is "for." Observe and oversight are the canonical examples: observe enforces "no obligations created" via `allowedTypes = {EVENT}`; oversight enforces "no telemetry" via `deniedTypes = {EVENT}`. Channels that participate in the full commitment lifecycle should have `allowedTypes = null`.

---

## Scope and cross-repo impact

- **Qhorus:** migration + `StoredMessageTypePolicy` + API surface (MCP tool param + ChannelDetail response)
- **Claudony:** `ChannelSpec` record + `NormativeChannelLayout` + `ClaudonyCaseChannelProvider` + tests + spec doc
- **casehub-openclaw:** aligns its own layout to the Claudony decision (their issue openclaw#6)
- **PLATFORM.md:** update oversight description from `"COMMAND → human, RESPONSE from human"` to reflect the full type model

---

## What is not changing

- `SimpleLayout` (no oversight channel — unaffected)
- The three-channel structure itself
- The commitment state machine in Qhorus
- The `allowedTypes` mechanism for `observe` (EVENT-only, unchanged)
