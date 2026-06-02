# Design: Oversight Channel `allowedTypes` — `deniedTypes` + Full Commitment Lifecycle
**Issue:** claudony#142  
**Date:** 2026-06-02  
**Status:** Approved — post-review revision

---

## Problem

`NormativeChannelLayout` sets `oversight.allowedTypes = {QUERY, COMMAND}`. This blocks six concrete type flows:

1. `RESPONSE` and `DECLINE` — engine gate (engine#402) cannot close approval commitments
2. `STATUS` — humans cannot extend deadlines on oversight commitments
3. `DONE` and `FAILURE` — agents cannot discharge COMMANDs issued by humans
4. `HANDOFF` — humans cannot delegate governance decisions
5. Watchdog deadline EVENTs — blocked (though these should route to `observe` anyway — see §Pattern A)
6. `RESPONSE` on oversight is shown in the mesh framework spec at line 300 but blocked by its line 708 — the spec has been internally inconsistent from the start

Root cause: `allowedTypes` was set as a categorisation label ("this is a governance channel") rather than a negative constraint ("this type must never appear here").

---

## Decision

### The invariant

Every channel in the three-channel layout enforces exactly one architectural constraint:

| Channel | Invariant | Mechanism |
|---------|-----------|-----------|
| `observe` | No obligations created | `allowedTypes = {EVENT}` |
| `oversight` | No pure telemetry | `deniedTypes = {EVENT}` |
| `work` | No constraint | `allowedTypes = null` |

`observe` and `oversight` are reciprocal: observe says "nothing that creates obligations"; oversight says "nothing that creates no obligations." `work` is the open coordination space.

### Why `deniedTypes` over enumerating 8 types (Option 1)

A new obligation-carrying type added to Qhorus's `MessageType` enum is automatically permitted on oversight without any Claudony change. Option 1 (enumeration) would silently block it.

### Why `deniedTypes` over channel roles (Option 3)

Option 3 would define `GOVERNANCE` as a Qhorus concept with a built-in type mapping. **This is the wrong layer.** Qhorus is a generic message substrate — it should not model domain semantics like "governance." `deniedTypes = {EVENT}` is a filter mechanism with no domain meaning; Claudony decides what goes in it, and Qhorus enforces it without understanding why. A different consumer with a different governance model gets a different `deniedTypes` set. Under Option 3, any consumer needing a different policy must add an override mechanism — which collapses back into `deniedTypes`. The right place for governance semantics is in Claudony; the right role for Qhorus is enforcement.

**Forward-compatibility caveat:** if Qhorus adds a future type with no commitment effect (e.g. `HEARTBEAT`), it would slip through `deniedTypes = {EVENT}` and appear on oversight. `MessageType` is a Qhorus API enum — any addition is a deliberate change requiring a Qhorus release, not a silent one. But "deliberate" alone doesn't prevent it being missed. The obligation is mechanical: when a new `MessageType` is added, any type with no commitment effect must be added to `deniedTypes` on all governance channels. To make this traceable, the `NormativeChannelLayout` constant must carry a comment naming this obligation explicitly (see implementation §3).

### Why EVENT is excluded from oversight

- `EVENT` is "observer-only telemetry — NOT delivered to agent context" (MessageType Javadoc)
- `pollAfter()` excludes `EVENT` by default — governance participants cannot see EVENTs on oversight
- Watchdog alerts have a configurable `notificationChannel`; they should route to `observe` where they are visible, not oversight where they are invisible to participants
- "Oversight is a governance space, not a progress log" maps to "no telemetry" — not "restricted speech acts"

### Nested commitments are supported

Qhorus `CommitmentService` is keyed by `correlationId` (UUID), not by channel. Two OPEN commitments with different `correlationId`s on the same channel are fully supported by the current model. Pattern A's nested QUERY is valid.

---

## Full required type set

Every realistic governance pattern traced through the Qhorus commitment state machine (see `design/oversight-type-map.md`):

`COMMAND, QUERY, RESPONSE, DONE, DECLINE, FAILURE, STATUS, HANDOFF`

Equivalently: all types except `EVENT`.

---

## Governance patterns — actor clarification

### Pattern A: Agent asks human for input
The Claude Code agent session posts `QUERY` to oversight. Humans respond with `RESPONSE` or `DECLINE`. Watchdog alerts about this commitment should be registered with `notificationChannel = observe` (not oversight), since EVENTs are excluded from oversight and invisible to participants there.

### Pattern B: Engine gate — ActionRiskClassifier (engine#402)
**Actor clarification:** the engine does not write to channels. When `ActionRiskClassifier.classify()` returns `GateRequired`, the engine pauses workflow execution. The Claude Code agent session — notified of the gate state via `check_messages` — posts `COMMAND` to the oversight channel requesting human approval. The human posts `RESPONSE` or `DECLINE` to close the commitment. How the engine detects commitment resolution (to resume the paused workflow) is part of engine#402's design, not specified here.

### Pattern C: Human injects directive
Human posts `COMMAND`; agent responds with `DONE` or `FAILURE`.

### Pattern D: Delegation — known limitation
`HANDOFF` transfers obligation to a named target. On an oversight channel, the target is a human instance ID. Whether a second human who was not an original channel participant receives notifications depends on Qhorus channel membership semantics, which are not fully specified. This is a known limitation: `HANDOFF` is a required type for the oversight channel to not block it, but the multi-party targeting semantics are an open design question, tracked as future work.

---

## Implementation

### Qhorus changes

**Migration — two steps:**

Step 1 — add `denied_types` column:
```sql
ALTER TABLE channel ADD COLUMN denied_types TEXT;
```
Format: comma-separated `MessageType` names, same serialization as `allowed_types`. Null means no denial.

Step 2 — fix existing oversight channels (data migration, same Flyway script or next version):
```sql
UPDATE channel
SET allowed_types = NULL,
    denied_types  = 'EVENT'
WHERE name LIKE '%/oversight';
```
This targets channels by name pattern. Existing oversight channels created before the fix had `allowed_types = 'COMMAND,QUERY'`; this clears that restriction and sets the new policy.

**`StoredMessageTypePolicy`:**
- Check `deniedTypes` after `allowedTypes`. Denied types are rejected regardless of `allowedTypes`. Priority: denial always wins.
- Creation-time validation: `create_channel` must **reject with an error** (HTTP 400 / exception) when `allowedTypes` and `deniedTypes` are both non-null and intersect. The channel is not created. A log warning without rejection adds nothing — "deny wins" is already the runtime behaviour; the validation's only purpose is to prevent a misconfigured channel from being persisted in the first place. Callers are responsible for never constructing an overlapping set; the API enforces this as a hard invariant, not a best-effort check.

**`ReactiveChannelService.create()` — Java API change:**
The existing 9-parameter signature gains `deniedTypes` as a 10th parameter (String, nullable, comma-separated). This is the primary integration point for Claudony and must be updated before any Claudony changes can compile.

**MCP tool `create_channel`:** new `denied_types` parameter (string, nullable).

**`ChannelDetail` response:** new `deniedTypes` field (string, nullable).

**Qhorus test coverage required:**
- `StoredMessageTypePolicy` correctly rejects a denied type even when `allowedTypes` is null
- Denied type is rejected even when it also appears in `allowedTypes` (deny wins)
- `create_channel` MCP tool stores `denied_types` and round-trips it in `ChannelDetail`
- `create_channel` returns an error when `allowedTypes` and `deniedTypes` intersect

---

### Claudony changes

**`CaseChannelLayout.ChannelSpec` record — source-breaking change:**

```java
record ChannelSpec(
    String purpose,
    ChannelSemantic semantic,
    Set<MessageType> allowedTypes,  // null = open
    Set<MessageType> deniedTypes,   // null = no denial
    String description
)
```

This changes the canonical constructor from 4-argument to 5-argument. Every construction site fails to compile. Mandatory update sites:
- `NormativeChannelLayout` (primary change)
- `SimpleLayout` (mechanical: add `null` for `deniedTypes`)
- `casehub-openclaw`'s `CaseChannelLayout` implementations — mandatory compiler-level dependency, not an optional follow-on. The openclaw team must update simultaneously. This is a coordinated cross-repo change.

**`NormativeChannelLayout`:**
```java
new ChannelSpec("oversight", ChannelSemantic.APPEND,
    null,                          // allowedTypes: open — all obligation-carrying types permitted
    // If a new MessageType is added to Qhorus with no commitment effect (like EVENT),
    // add it here. This comment is the mechanical anchor for that obligation.
    Set.of(MessageType.EVENT),     // deniedTypes: no telemetry on the governance channel
    "Human governance — all obligation-carrying types; no telemetry"),
new ChannelSpec("work", ChannelSemantic.APPEND,
    null, null,
    "Primary coordination — all obligation-carrying message types"),
new ChannelSpec("observe", ChannelSemantic.APPEND,
    Set.of(MessageType.EVENT),
    null,                          // deniedTypes null: EVENT is already the only allowed type, denial is redundant
    "Telemetry — EVENT only, no obligations created")
```

**`ClaudonyCaseChannelProvider` — serialization:**
Add `toDeniedTypesString(Set<MessageType> deniedTypes)` alongside the existing `toAllowedTypesString()`. Same format: comma-separated type names, null input → null output. Pass result as the 10th argument to `ReactiveChannelService.create()`.

**`NormativeChannelLayoutTest`:**
Replace the assertion `assertThat(oversight.allowedTypes()).containsExactlyInAnyOrder(MessageType.QUERY, MessageType.COMMAND)` with:
- `assertThat(oversight.allowedTypes()).isNull()`
- `assertThat(oversight.deniedTypes()).containsExactly(MessageType.EVENT)`

**`ClaudonyReactiveCaseChannelProviderTest` — extensive updates:**
Every `when(channelService.create(...))` and `verify(channelService.create(...))` stub uses 9-argument Mockito matchers. The 10th `deniedTypes` parameter must be added to every call site. Specific assertion at line 209–211 that asserts `eq("COMMAND,QUERY")` as the `allowedTypes` argument becomes:
- `allowedTypes` argument: `isNull()`
- `deniedTypes` argument (new): `eq("EVENT")`

**Cache interaction with live channels:**
`ClaudonyReactiveCaseChannelProvider.openChannel()` caches by `caseId`. If an oversight channel already exists in Qhorus (created before the fix), `channelService.create()` returns the existing channel without updating its policy. The data migration (SQL UPDATE above) handles this at the DB level for all existing rows — this is sufficient for correctness. The Qhorus create-or-find path will see the already-updated `denied_types` on the existing channel.

---

### Documentation and protocol

**Mesh framework spec** (`docs/specs/2026-04-27-claudony-agent-mesh-framework.md`):

Line 708 — change to:
> **`allowed_types` / `denied_types`** — Pass `"EVENT"` as `allowed_types` when creating the observe channel (telemetry only, no obligations). Pass `"EVENT"` as `denied_types` when creating the oversight channel (all obligation-carrying types permitted; no telemetry). Work channel: both null (open). Enforced server-side by `StoredMessageTypePolicy`.

Line 68 — update oversight description to:
> **`oversight`** — The human governance channel. Agents post `COMMAND` here when requesting human approval for consequential actions; `QUERY` when seeking human input. Humans post `RESPONSE`, `DECLINE`, `DONE`, `STATUS`, or `HANDOFF` as appropriate. The full commitment lifecycle is supported. `EVENT` is excluded — governance conversations must not contain telemetry; Watchdog alerts about oversight commitments should register `notificationChannel = observe`.

Line 593 — update the oversight row:
```
oversight  APPEND  deniedTypes={EVENT}  human governance (all obligation-carrying types)
```

**`design/oversight-type-map.md`** — update the Channel invariants table to replace Option 1 syntax with Option 2:
```
oversight | No pure telemetry | deniedTypes = EVENT
```
Remove the Option 1 row from the Implementation Options section (it was the "enumerate 8 types" approach, now superseded).

**PLATFORM.md** — update oversight description:
Replace: `| oversight | Human governance gates (commitment-based) | COMMAND → human, RESPONSE from human |`
With: `| oversight | Human governance gates (commitment-based) | All obligation-carrying types (COMMAND, QUERY, RESPONSE, DONE, DECLINE, FAILURE, STATUS, HANDOFF); EVENT excluded — no telemetry on the governance channel |`

**Protocol** — capture via `protocol` skill in the parent repo (`casehubio/parent/docs/protocols/casehub/`):

> **`allowedTypes` is an architectural invariant, not a category label.** Only set `allowedTypes` or `deniedTypes` when there is a hard constraint that must be enforced across all scenarios. The `observe` channel enforces "no obligations created" via `allowedTypes = {EVENT}`. The `oversight` channel enforces "no telemetry" via `deniedTypes = {EVENT}`. Channels that participate in the full commitment lifecycle should have `allowedTypes = null`. Contradictory configurations (`allowedTypes` and `deniedTypes` overlapping) must be rejected at channel creation time.

---

## Scope and cross-repo impact

| Repo | Changes | Blocking? |
|------|---------|-----------|
| `casehub-qhorus` | DB migration (2 steps), `StoredMessageTypePolicy`, `ReactiveChannelService.create()`, MCP tool, `ChannelDetail`, tests | Yes — Claudony won't compile without it |
| `claudony` | `ChannelSpec` record, `NormativeChannelLayout`, `SimpleLayout`, `ClaudonyCaseChannelProvider` + `toDeniedTypesString()`, all test updates | — |
| `casehub-openclaw` | Update all `CaseChannelLayout` implementations — mandatory, compiler-level | Yes — breaking record change |
| `casehub-parent` | Protocol file, `PLATFORM.md` | No, but should ship before or alongside |

**What is not changing:**
- `SimpleLayout` loses oversight — unaffected by content, mechanical update only for `ChannelSpec` constructor
- Commitment state machine in Qhorus
- `allowedTypes` mechanism for `observe` (EVENT-only, unchanged)
- Three-channel structure
