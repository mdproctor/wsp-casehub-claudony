# Promote Task/Correlation/Artifact Panels to blocks-ui

**Issue:** casehubio/claudony#180
**Date:** 2026-07-21
**Status:** Approved

## Summary

Move three LitElement panels (`claudony-task-panel`, `claudony-correlation-panel`, `claudony-artifact-panel`) from Claudony's webui into `@casehubio/blocks-ui-channel-activity`. Rename from `claudony-*` to `channel-*` prefix. Move `CommitmentRecord` type and conversion functions alongside them.

## Scope

Two repos change:
- **blocks-ui** (`/Users/mdproctor/claude/casehub/blocks-ui`) — receives the panels
- **claudony** (`/Users/mdproctor/claude/casehub/claudony`) — removes them, updates imports

## blocks-ui Changes

### types.ts — changes

Make `topicId` optional on `QhorusMessage` (`topicId?: string`) — it's only meaningful when topic mode is active, and existing consumers (claudony's `toQhorusMessage`) don't set it.

Add `ResolvedArtifact` interface for the artifact panel's callback return type:

```typescript
export interface ResolvedArtifact {
  readonly content: string;
  readonly language?: string;
}
```

### commitment.ts — new module

New file `components/channel-activity/src/commitment.ts` groups commitment types and conversion logic, keeping `types.ts` as the Qhorus domain vocabulary (messages, channels, topics, artefacts).

```typescript
import type { CommitmentState } from './types.js';

export interface RawCommitment {
  id: string;
  correlationId: string;
  state: string;
  requester?: string;
  obligor?: string;
  expiresAt?: string | null;
  acknowledgedAt?: string | null;
  resolvedAt?: string | null;
  createdAt?: string | null;
}

export interface CommitmentRecord {
  readonly state: CommitmentState;
  readonly deadline?: string;
  readonly acknowledgedAt?: string;
  readonly createdAt: string;
  readonly updatedAt: string;
}

export function toCommitmentRecord(raw: RawCommitment): CommitmentRecord;
export function toCommitmentMap(commitments: RawCommitment[]): Map<string, CommitmentRecord>;
```

`toCommitmentMap` keys by `correlationId` — the contract is that `commitment.correlationId` equals the `id` of the COMMAND message that created it. Panels look up commitments by `msg.id`, which matches because the engine sets `correlationId = String(eventLogId)` and the message id is the same eventLogId.

**CommitmentState vs CommitmentRecord**: `channel-feed` uses `Map<string, CommitmentState>` (lightweight — only needs the state string for badge rendering). The new panels use `Map<string, CommitmentRecord>` (needs timestamps for deadline checking and time display). This divergence is intentional — `CommitmentRecord.state` is a `CommitmentState`.

### New components

Three new files, following existing channel-activity naming:

| File | Element tag | Class |
|------|-------------|-------|
| `channel-task-panel.ts` | `<channel-task-panel>` | `ChannelTaskPanelElement` |
| `channel-correlation-panel.ts` | `<channel-correlation-panel>` | `ChannelCorrelationPanelElement` |
| `channel-artifact-panel.ts` | `<channel-artifact-panel>` | `ChannelArtifactPanelElement` |

Each gets a corresponding `.test.ts` file following the existing vitest + jsdom pattern (see `channel-member-panel.test.ts` for reference).

Code is the Claudony source with:
- Tag names: `claudony-*` → `channel-*`
- Class names: `Claudony*Element` → `Channel*Element`
- Package self-imports → relative internal imports (all three panels import from `@casehubio/blocks-ui-channel-activity` which becomes a self-reference inside the package):
  - `QhorusMessage`, `commitmentStateCategory`, `isObligationCreating`, `messageTypeCategory`: `@casehubio/blocks-ui-channel-activity` → `./types.js`
  - `ChannelEventTopics`: `@casehubio/blocks-ui-channel-activity` → `./events.js`
  - `ArtefactRef` (artifact panel only): `@casehubio/blocks-ui-channel-activity` → `./types.js`
- `CommitmentRecord` import: `../util/channel-adapter.js` → `./commitment.js`
- `emitPagesEvent` from `@casehubio/blocks-ui-core` stays unchanged — cross-package dependency, not self-referencing
- CSS variables: strip all inline fallback values to bare `var(--token)` form, matching blocks-ui's existing convention (e.g. `channel-member-panel`). The host app provides tokens; component-level fallbacks mask configuration problems.
- Artifact panel: use `ResolvedArtifact` named type for `resolveArtifact` callback return instead of anonymous inline `{ content: string; language?: string }`

### index.ts

Add exports for all three element classes + `RawCommitment` + `CommitmentRecord` + `toCommitmentRecord` + `toCommitmentMap` (re-exported from `commitment.ts`) + `ResolvedArtifact`.

### Version

`0.2.2` → `0.3.0` (new public API surface).

## Claudony Changes

### Delete

- `app/src/main/webui/src/components/claudony-task-panel.ts`
- `app/src/main/webui/src/components/claudony-correlation-panel.ts`
- `app/src/main/webui/src/components/claudony-artifact-panel.ts`

### channel-adapter.ts

Remove: `CommitmentRecord` interface, `RawCommitment` interface, `toCommitmentRecord`, `toCommitmentMap`.

Keep: `TimelineEntry`, `ChannelInfo`, `toQhorusMessage`, `toQhorusChannel`, `formatEventContent`.

### channel-adapter.test.ts

Remove tests for `toCommitmentRecord` and `toCommitmentMap` (they move to blocks-ui).

### claudony-workbench.ts

- Split the `channel-adapter.js` import:
  - Keep in `channel-adapter.js`: `toQhorusMessage`, `toQhorusChannel`, type `TimelineEntry`, type `ChannelInfo`
  - Import from `@casehubio/blocks-ui-channel-activity`: `CommitmentRecord`, `toCommitmentMap`
- Replace side-effect imports of the three panel files with imports from `@casehubio/blocks-ui-channel-activity`
- Template tags: `claudony-task-panel` → `channel-task-panel`, etc.

### package.json

Bump `@casehubio/blocks-ui-channel-activity` dependency from `^0.2.2` to `^0.3.0`.

### E2E tests

No changes needed — `WorkbenchE2ETest` selectors reference `claudony-workbench`, `#terminal-container`, and `.dock-btn`, none of which are affected by the panel rename.

### CLAUDE.md

Update project structure section — remove the three panel component entries, note they moved to blocks-ui.

## Showcase (blocks-ui examples app)

Add the three panels to the existing `channel-activity-page.ts` showcase page. The page already demonstrates feed, nav, input, member panel, and topic bar — the new panels are their natural companions.

**New mock data must use valid `MessageType` values** (`COMMAND`, `RESPONSE`, `STATUS`, `DONE`, `FAILURE`, `DECLINE`, `HANDOFF`, `EVENT`) and valid `ChannelSemantic` values. The existing showcase data uses invalid types like `'HUMAN'` and `'INFORM'` — this is pre-existing technical debt, not something the new sections should reproduce. The task panel depends on `isObligationCreating` which checks for `'COMMAND'`; the correlation panel needs real speech-act sequences; both would render empty with invalid types.

### Task panel demo

Mock data with COMMAND-type messages and commitment records covering all three groups:
- **Active** — open commitments with no deadline breach
- **Overdue** — open commitments past deadline
- **Completed** — terminal states (FULFILLED, FAILED, DECLINED, DELEGATED)

Include a timed data source that adds new commitments and transitions existing ones through states, demonstrating the panel updating live.

### Correlation panel demo

Mock data with a correlationId chain (3-4 messages in a command→status→done sequence) and an inReplyTo chain. Pre-select a message to show the chain on load.

### Artifact panel demo

Mock data with messages carrying artefactRefs of varying types (DOCUMENT, CODE, CASE, EXTERNAL). Wire the `resolveArtifact` callback to return mock content. Demonstrate back/forward navigation by selecting different artifacts.

### Shell registration

Add a new nav entry in `shell.ts` under Components — or extend the existing `channel-activity` page with new sections (simpler, keeps related components together). Extend the existing page — no separate page needed.

## Testing

- **blocks-ui:** New vitest tests for each panel (rendering, state grouping, click events, empty states, navigation history for artifact panel) + `commitment.test.ts` for `toCommitmentRecord` and `toCommitmentMap` (3 tests covering basic mapping, `updatedAt` derivation, and `correlationId` keying — migrated from claudony's `channel-adapter.test.ts`)
- **Claudony vitest:** Existing `channel-adapter.test.ts` tests shrink (commitment tests removed)
- **Claudony E2E:** `WorkbenchE2ETest` passes unchanged (selectors don't reference panel tag names)
- **Claudony Java tests:** No changes expected — panels are frontend-only

## Execution Order

1. blocks-ui: add types, components, tests, showcase sections, bump version, build + publish
2. Claudony: bump dependency, delete panels, update imports/templates/tests
