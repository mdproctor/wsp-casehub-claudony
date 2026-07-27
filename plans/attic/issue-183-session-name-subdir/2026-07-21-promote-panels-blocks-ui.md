# Promote Panels to blocks-ui Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #180 — chore: promote task/correlation/artifact panels to blocks-ui
**Issue group:** #180

**Goal:** Move three LitElement panels from Claudony into `@casehubio/blocks-ui-channel-activity`, rename from `claudony-*` to `channel-*`, and migrate `CommitmentRecord` types alongside them.

**Architecture:** Two repos change in sequence. blocks-ui receives the panels, types, commitment module, tests, and showcase demos. Then Claudony removes its copies, updates imports, and bumps the dependency. All panels are LitElement web components using `--pages-*` design tokens and `emitPagesEvent` from blocks-ui-core.

**Tech Stack:** TypeScript, Lit 3, vitest + jsdom, `@casehubio/blocks-ui-core`, `@casehubio/blocks-ui-channel-activity`

## Global Constraints

- blocks-ui CSS convention: bare `var(--pages-token)` — no inline fallback values
- Panel imports inside the package use relative paths (`./types.js`, `./events.js`, `./commitment.js`) — never self-referencing `@casehubio/blocks-ui-channel-activity`
- `emitPagesEvent` from `@casehubio/blocks-ui-core` stays as a cross-package import
- New mock data in showcase must use valid `MessageType` values (`COMMAND`, `RESPONSE`, `STATUS`, `DONE`, `FAILURE`, `DECLINE`, `HANDOFF`, `EVENT`)
- IntelliJ MCP required for all `.ts` file operations. Project path: `/Users/mdproctor/claude/casehub/blocks-ui` for blocks-ui, `/Users/mdproctor/claude/casehub/claudony` for Claudony.
- Test runner: `npx vitest run` from `components/channel-activity/` for blocks-ui; `npm --prefix app/src/main/webui test` for Claudony vitest.

---

### Task 1: Foundation types + commitment module (blocks-ui)

**Files:**
- Modify: `components/channel-activity/src/types.ts` (line 55: topicId, add ResolvedArtifact)
- Create: `components/channel-activity/src/commitment.ts`
- Create: `components/channel-activity/src/commitment.test.ts`

**Interfaces:**
- Produces: `CommitmentRecord`, `RawCommitment`, `toCommitmentRecord()`, `toCommitmentMap()`, `ResolvedArtifact` — used by Tasks 2-5

- [ ] **Step 1: Write commitment.test.ts**

Create `components/channel-activity/src/commitment.test.ts`:

```typescript
import { describe, it, expect } from 'vitest';
import { toCommitmentRecord, toCommitmentMap } from './commitment.js';
import type { RawCommitment } from './commitment.js';

describe('commitment', () => {
  it('maps commitment response to CommitmentRecord', () => {
    const raw: RawCommitment = {
      id: 'uuid-1', correlationId: 'corr-1', state: 'OPEN',
      requester: 'human:alice', obligor: 'agent-1',
      expiresAt: '2026-08-01T00:00:00Z', acknowledgedAt: null,
      resolvedAt: null, createdAt: '2026-07-19T10:00:00Z',
    };
    const record = toCommitmentRecord(raw);
    expect(record.state).toBe('OPEN');
    expect(record.deadline).toBe('2026-08-01T00:00:00Z');
    expect(record.createdAt).toBe('2026-07-19T10:00:00Z');
    expect(record.updatedAt).toBe('2026-07-19T10:00:00Z');
  });

  it('updatedAt is max of resolvedAt, acknowledgedAt, createdAt', () => {
    const raw: RawCommitment = {
      id: 'uuid-2', correlationId: 'corr-2', state: 'FULFILLED',
      requester: 'human:alice', obligor: 'agent-1',
      acknowledgedAt: '2026-07-19T11:00:00Z',
      resolvedAt: '2026-07-19T12:00:00Z',
      createdAt: '2026-07-19T10:00:00Z',
    };
    const record = toCommitmentRecord(raw);
    expect(record.updatedAt).toBe('2026-07-19T12:00:00Z');
  });

  it('toCommitmentMap keys by correlationId', () => {
    const commitments: RawCommitment[] = [
      { id: 'u1', correlationId: 'c1', state: 'OPEN', createdAt: '2026-01-01T00:00:00Z' },
      { id: 'u2', correlationId: 'c2', state: 'FULFILLED', createdAt: '2026-01-02T00:00:00Z' },
    ];
    const map = toCommitmentMap(commitments);
    expect(map.size).toBe(2);
    expect(map.get('c1')?.state).toBe('OPEN');
    expect(map.get('c2')?.state).toBe('FULFILLED');
  });

  it('handles null optional fields', () => {
    const raw: RawCommitment = {
      id: 'uuid-3', correlationId: 'corr-3', state: 'OPEN',
      expiresAt: null, acknowledgedAt: null, resolvedAt: null, createdAt: null,
    };
    const record = toCommitmentRecord(raw);
    expect(record.deadline).toBeUndefined();
    expect(record.acknowledgedAt).toBeUndefined();
    expect(record.createdAt).toBeDefined();
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `cd /Users/mdproctor/claude/casehub/blocks-ui/components/channel-activity && npx vitest run src/commitment.test.ts`
Expected: FAIL — `./commitment.js` does not exist

- [ ] **Step 3: Create commitment.ts**

Create `components/channel-activity/src/commitment.ts`:

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

export function toCommitmentRecord(raw: RawCommitment): CommitmentRecord {
  const timestamps = [raw.resolvedAt, raw.acknowledgedAt, raw.createdAt]
    .filter((t): t is string => t != null);
  const updatedAt = timestamps.length > 0
    ? timestamps.reduce((a, b) => a > b ? a : b)
    : raw.createdAt ?? new Date().toISOString();

  return {
    state: raw.state as CommitmentState,
    deadline: raw.expiresAt ?? undefined,
    acknowledgedAt: raw.acknowledgedAt ?? undefined,
    createdAt: raw.createdAt ?? new Date().toISOString(),
    updatedAt,
  };
}

export function toCommitmentMap(
  commitments: RawCommitment[]
): Map<string, CommitmentRecord> {
  const map = new Map<string, CommitmentRecord>();
  for (const c of commitments) {
    map.set(c.correlationId, toCommitmentRecord(c));
  }
  return map;
}
```

- [ ] **Step 4: Make topicId optional and add ResolvedArtifact in types.ts**

In `components/channel-activity/src/types.ts`, change line 55 from:
```
  readonly topicId: string;
```
to:
```
  readonly topicId?: string;
```

Add after the `ArtefactRef` interface (after line 40):
```typescript
export interface ResolvedArtifact {
  readonly content: string;
  readonly language?: string;
}
```

- [ ] **Step 5: Run tests to verify they pass**

Run: `cd /Users/mdproctor/claude/casehub/blocks-ui/components/channel-activity && npx vitest run`
Expected: ALL PASS (including the 3 new commitment tests + all existing tests)

- [ ] **Step 6: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/blocks-ui add components/channel-activity/src/commitment.ts components/channel-activity/src/commitment.test.ts components/channel-activity/src/types.ts
git -C /Users/mdproctor/claude/casehub/blocks-ui commit -m "feat(#180): add commitment module, ResolvedArtifact, optional topicId

Refs casehubio/claudony#180"
```

---

### Task 2: Channel task panel (blocks-ui)

**Files:**
- Create: `components/channel-activity/src/channel-task-panel.ts`
- Create: `components/channel-activity/src/channel-task-panel.test.ts`

**Interfaces:**
- Consumes: `QhorusMessage`, `commitmentStateCategory`, `isObligationCreating` from `./types.js`; `ChannelEventTopics` from `./events.js`; `CommitmentRecord` from `./commitment.js`; `emitPagesEvent` from `@casehubio/blocks-ui-core`
- Produces: `<channel-task-panel>` element, `ChannelTaskPanelElement` class

- [ ] **Step 1: Write channel-task-panel.test.ts**

Create `components/channel-activity/src/channel-task-panel.test.ts`:

```typescript
import { describe, it, expect, afterEach } from 'vitest';
import './channel-task-panel.js';
import type { QhorusMessage } from './types.js';
import type { CommitmentRecord } from './commitment.js';

function makeMsg(overrides: Partial<QhorusMessage> = {}): QhorusMessage {
  return {
    id: 'm1', channelId: 'ch-1', sender: 'human:alice', messageType: 'COMMAND',
    actorType: 'HUMAN', content: 'Run the compliance check', topic: '',
    replyCount: 0, artefactRefs: [], createdAt: '2026-07-20T10:00:00Z',
    ...overrides,
  };
}

describe('channel-task-panel', () => {
  let element: HTMLElement;

  afterEach(() => { element?.remove(); });

  it('renders empty state when no commands', async () => {
    element = document.createElement('channel-task-panel') as any;
    (element as any).messages = [
      makeMsg({ id: 'm1', messageType: 'STATUS' }),
    ];
    (element as any).commitments = new Map();
    document.body.appendChild(element);
    await (element as any).updateComplete;

    expect(element.shadowRoot!.querySelector('.empty')?.textContent).toContain('No commitments');
  });

  it('groups tasks into active, overdue, completed', async () => {
    const past = new Date(Date.now() - 86400000).toISOString();
    const future = new Date(Date.now() + 86400000).toISOString();

    element = document.createElement('channel-task-panel') as any;
    (element as any).messages = [
      makeMsg({ id: 'active', content: 'Active task' }),
      makeMsg({ id: 'overdue', content: 'Overdue task' }),
      makeMsg({ id: 'done', content: 'Done task' }),
    ];
    const commitments = new Map<string, CommitmentRecord>([
      ['active', { state: 'OPEN', deadline: future, createdAt: '2026-07-20T10:00:00Z', updatedAt: '2026-07-20T10:00:00Z' }],
      ['overdue', { state: 'OPEN', deadline: past, createdAt: '2026-07-20T10:00:00Z', updatedAt: '2026-07-20T10:00:00Z' }],
      ['done', { state: 'FULFILLED', createdAt: '2026-07-20T10:00:00Z', updatedAt: '2026-07-20T11:00:00Z' }],
    ]);
    (element as any).commitments = commitments;
    document.body.appendChild(element);
    await (element as any).updateComplete;

    const groups = element.shadowRoot!.querySelectorAll('.group-label');
    const labels = Array.from(groups).map(g => g.textContent?.trim());
    expect(labels).toContain('Overdue');
    expect(labels).toContain('Active');
    expect(labels).toContain('Completed');
  });

  it('marks overdue rows with overdue class', async () => {
    const past = new Date(Date.now() - 86400000).toISOString();
    element = document.createElement('channel-task-panel') as any;
    (element as any).messages = [makeMsg({ id: 'od' })];
    (element as any).commitments = new Map([
      ['od', { state: 'OPEN', deadline: past, createdAt: '2026-07-20T10:00:00Z', updatedAt: '2026-07-20T10:00:00Z' }],
    ]);
    document.body.appendChild(element);
    await (element as any).updateComplete;

    const row = element.shadowRoot!.querySelector('.task-row');
    expect(row?.classList.contains('overdue')).toBe(true);
  });

  it('emits message-selected event on row click', async () => {
    element = document.createElement('channel-task-panel') as any;
    const msg = makeMsg({ id: 'click-me' });
    (element as any).messages = [msg];
    (element as any).commitments = new Map();
    document.body.appendChild(element);
    await (element as any).updateComplete;

    let emitted: any = null;
    element.addEventListener('pages-event', (e: Event) => { emitted = (e as CustomEvent).detail; });

    const row = element.shadowRoot!.querySelector('.task-row') as HTMLElement;
    row.click();

    expect(emitted).not.toBeNull();
    expect(emitted.payload.message.id).toBe('click-me');
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `cd /Users/mdproctor/claude/casehub/blocks-ui/components/channel-activity && npx vitest run src/channel-task-panel.test.ts`
Expected: FAIL — `./channel-task-panel.js` does not exist

- [ ] **Step 3: Create channel-task-panel.ts**

Create `components/channel-activity/src/channel-task-panel.ts` — adapted from Claudony's `claudony-task-panel.ts` with these changes:
- Tag: `channel-task-panel`, Class: `ChannelTaskPanelElement`
- Imports: `QhorusMessage`, `commitmentStateCategory`, `isObligationCreating` from `./types.js`; `ChannelEventTopics` from `./events.js`; `CommitmentRecord` from `./commitment.js`; `emitPagesEvent` from `@casehubio/blocks-ui-core`
- All CSS `var(--pages-*, fallback)` → `var(--pages-*)`
- HTMLElementTagNameMap declares `'channel-task-panel'`

```typescript
import { LitElement, html, css, nothing } from 'lit';
import { customElement, property } from 'lit/decorators.js';
import type { QhorusMessage } from './types.js';
import { commitmentStateCategory, isObligationCreating } from './types.js';
import { emitPagesEvent } from '@casehubio/blocks-ui-core';
import { ChannelEventTopics } from './events.js';
import type { CommitmentRecord } from './commitment.js';

@customElement('channel-task-panel')
export class ChannelTaskPanelElement extends LitElement {
  @property({ type: Array }) messages: QhorusMessage[] = [];
  @property({ type: Object }) commitments: Map<string, CommitmentRecord> = new Map();
  @property({ type: String }) selectedMessageId?: string;

  static override readonly styles = css`
    :host {
      display: flex;
      flex-direction: column;
      height: 100%;
      overflow-y: auto;
      font-family: var(--pages-font-family);
    }
    .panel-title {
      font-size: var(--pages-font-size-sm);
      font-weight: var(--pages-font-weight-semibold);
      padding: var(--pages-space-3) var(--pages-space-4);
      border-bottom: 1px solid var(--pages-neutral-4);
      color: var(--pages-neutral-12);
    }
    .group-label {
      font-size: var(--pages-font-size-xs);
      font-weight: var(--pages-font-weight-medium);
      color: var(--pages-neutral-8);
      text-transform: uppercase;
      letter-spacing: 0.5px;
      padding: var(--pages-space-2) var(--pages-space-4) var(--pages-space-1);
    }
    .task-row {
      display: flex;
      flex-direction: column;
      gap: var(--pages-space-1);
      padding: var(--pages-space-2) var(--pages-space-4);
      cursor: pointer;
      border-bottom: 1px solid var(--pages-neutral-3);
    }
    .task-row:hover { background: var(--pages-neutral-2); }
    .task-row.selected { background: var(--pages-accent-2); }
    .task-row.overdue { border-left: 3px solid var(--pages-danger-9); }
    .task-header {
      display: flex;
      align-items: center;
      gap: var(--pages-space-2);
    }
    .state-badge {
      font-size: 10px;
      font-weight: var(--pages-font-weight-medium);
      padding: 1px 6px;
      border-radius: var(--pages-radius-sm);
      text-transform: uppercase;
    }
    .badge-active { background: var(--pages-accent-3); color: var(--pages-accent-11); }
    .badge-info { background: var(--pages-info-3); color: var(--pages-info-11); }
    .badge-success { background: var(--pages-success-3); color: var(--pages-success-11); }
    .badge-danger { background: var(--pages-danger-3); color: var(--pages-danger-11); }
    .badge-neutral { background: var(--pages-neutral-3); color: var(--pages-neutral-9); }
    .badge-transfer { background: var(--pages-info-3); color: var(--pages-info-11); }
    .badge-warning { background: var(--pages-warning-3); color: var(--pages-warning-11); }
    .sender-target {
      font-size: var(--pages-font-size-xs);
      color: var(--pages-neutral-9);
    }
    .content-preview {
      font-size: var(--pages-font-size-sm);
      color: var(--pages-neutral-11);
      overflow: hidden;
      text-overflow: ellipsis;
      white-space: nowrap;
    }
    .timestamp {
      font-size: var(--pages-font-size-xs);
      color: var(--pages-neutral-8);
    }
    .deadline-indicator {
      font-size: var(--pages-font-size-xs);
      color: var(--pages-danger-9);
      font-weight: var(--pages-font-weight-medium);
    }
    .terminal-group { opacity: 0.7; }
    .empty {
      display: flex;
      align-items: center;
      justify-content: center;
      height: 100%;
      color: var(--pages-neutral-8);
      font-size: var(--pages-font-size-sm);
    }
  `;

  private _commands(): QhorusMessage[] {
    return this.messages.filter(m => isObligationCreating(m.messageType));
  }

  private _isOverdue(record: CommitmentRecord | undefined): boolean {
    if (!record?.deadline) return false;
    return record.state === 'OPEN' && new Date(record.deadline) < new Date();
  }

  private _isTerminal(state: string): boolean {
    return ['FULFILLED', 'FAILED', 'DECLINED', 'DELEGATED', 'EXPIRED'].includes(state);
  }

  private _formatTime(iso: string): string {
    const date = new Date(iso);
    const now = new Date();
    const diffMs = now.getTime() - date.getTime();
    const diffMin = Math.floor(diffMs / 60000);
    if (diffMin < 1) return 'now';
    if (diffMin < 60) return `${diffMin}m`;
    const diffHr = Math.floor(diffMin / 60);
    if (diffHr < 24) return `${diffHr}h`;
    return `${Math.floor(diffHr / 24)}d`;
  }

  private _onRowClick(msg: QhorusMessage) {
    emitPagesEvent(this, ChannelEventTopics.MESSAGE_SELECTED, { message: msg });
  }

  override render() {
    const commands = this._commands();
    if (commands.length === 0) {
      return html`<div class="panel-title">Tasks</div><div class="empty">No commitments in this channel</div>`;
    }

    const active: QhorusMessage[] = [];
    const overdue: QhorusMessage[] = [];
    const terminal: QhorusMessage[] = [];

    for (const cmd of commands) {
      const record = this.commitments.get(cmd.id);
      const state = record?.state ?? 'OPEN';
      if (this._isOverdue(record)) {
        overdue.push(cmd);
      } else if (this._isTerminal(state)) {
        terminal.push(cmd);
      } else {
        active.push(cmd);
      }
    }

    return html`
      <div class="panel-title">Tasks</div>
      ${overdue.length > 0 ? html`
        <div class="group-label">Overdue</div>
        ${overdue.map(m => this._renderRow(m))}
      ` : nothing}
      ${active.length > 0 ? html`
        <div class="group-label">Active</div>
        ${active.map(m => this._renderRow(m))}
      ` : nothing}
      ${terminal.length > 0 ? html`
        <div class="group-label terminal-group">Completed</div>
        <div class="terminal-group">
          ${terminal.map(m => this._renderRow(m))}
        </div>
      ` : nothing}
    `;
  }

  private _renderRow(msg: QhorusMessage) {
    const record = this.commitments.get(msg.id);
    const state = record?.state ?? 'OPEN';
    const category = commitmentStateCategory(state as any);
    const isOverdue = this._isOverdue(record);
    const isSelected = this.selectedMessageId === msg.id;

    return html`
      <div class="task-row ${isOverdue ? 'overdue' : ''} ${isSelected ? 'selected' : ''}"
           @click=${() => this._onRowClick(msg)}>
        <div class="task-header">
          <span class="state-badge badge-${category}">${state}</span>
          <span class="timestamp">${this._formatTime(msg.createdAt)}</span>
          ${isOverdue && record?.deadline ? html`
            <span class="deadline-indicator">overdue</span>
          ` : nothing}
        </div>
        <div class="content-preview">${msg.content.split('\n')[0]}</div>
        <div class="sender-target">
          ${msg.sender}${msg.target ? html` → ${msg.target}` : nothing}
        </div>
      </div>
    `;
  }
}

declare global {
  interface HTMLElementTagNameMap {
    'channel-task-panel': ChannelTaskPanelElement;
  }
}
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `cd /Users/mdproctor/claude/casehub/blocks-ui/components/channel-activity && npx vitest run src/channel-task-panel.test.ts`
Expected: ALL 4 PASS

- [ ] **Step 5: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/blocks-ui add components/channel-activity/src/channel-task-panel.ts components/channel-activity/src/channel-task-panel.test.ts
git -C /Users/mdproctor/claude/casehub/blocks-ui commit -m "feat(#180): add channel-task-panel component

Refs casehubio/claudony#180"
```

---

### Task 3: Channel correlation panel (blocks-ui)

**Files:**
- Create: `components/channel-activity/src/channel-correlation-panel.ts`
- Create: `components/channel-activity/src/channel-correlation-panel.test.ts`

**Interfaces:**
- Consumes: `QhorusMessage`, `messageTypeCategory`, `commitmentStateCategory`, `isObligationCreating` from `./types.js`; `ChannelEventTopics` from `./events.js`; `CommitmentRecord` from `./commitment.js`; `emitPagesEvent` from `@casehubio/blocks-ui-core`
- Produces: `<channel-correlation-panel>` element, `ChannelCorrelationPanelElement` class

- [ ] **Step 1: Write channel-correlation-panel.test.ts**

Create `components/channel-activity/src/channel-correlation-panel.test.ts`:

```typescript
import { describe, it, expect, afterEach } from 'vitest';
import './channel-correlation-panel.js';
import type { QhorusMessage } from './types.js';

function makeMsg(overrides: Partial<QhorusMessage> = {}): QhorusMessage {
  return {
    id: 'm1', channelId: 'ch-1', sender: 'human:alice', messageType: 'COMMAND',
    actorType: 'HUMAN', content: 'Review the case', topic: '',
    replyCount: 0, artefactRefs: [], createdAt: '2026-07-20T10:00:00Z',
    ...overrides,
  };
}

describe('channel-correlation-panel', () => {
  let element: HTMLElement;

  afterEach(() => { element?.remove(); });

  it('shows empty state when no message selected', async () => {
    element = document.createElement('channel-correlation-panel') as any;
    (element as any).messages = [];
    (element as any).commitments = new Map();
    document.body.appendChild(element);
    await (element as any).updateComplete;

    expect(element.shadowRoot!.querySelector('.empty')?.textContent).toContain('Select a message');
  });

  it('builds chain from correlationId', async () => {
    element = document.createElement('channel-correlation-panel') as any;
    (element as any).messages = [
      makeMsg({ id: 'm1', correlationId: 'corr-1', sender: 'human:alice', messageType: 'COMMAND', createdAt: '2026-07-20T10:00:00Z' }),
      makeMsg({ id: 'm2', correlationId: 'corr-1', sender: 'agent-1', messageType: 'STATUS', createdAt: '2026-07-20T10:01:00Z' }),
      makeMsg({ id: 'm3', correlationId: 'corr-1', sender: 'agent-1', messageType: 'DONE', createdAt: '2026-07-20T10:02:00Z' }),
      makeMsg({ id: 'unrelated', correlationId: 'corr-2', sender: 'agent-2', messageType: 'STATUS' }),
    ];
    (element as any).commitments = new Map();
    (element as any).selectedMessageId = 'm1';
    document.body.appendChild(element);
    await (element as any).updateComplete;

    const nodes = element.shadowRoot!.querySelectorAll('.flow-node');
    expect(nodes.length).toBe(3);
  });

  it('builds chain from inReplyTo', async () => {
    element = document.createElement('channel-correlation-panel') as any;
    (element as any).messages = [
      makeMsg({ id: 'm1', sender: 'human:alice', messageType: 'COMMAND', createdAt: '2026-07-20T10:00:00Z' }),
      makeMsg({ id: 'm2', inReplyTo: 'm1', sender: 'agent-1', messageType: 'RESPONSE', createdAt: '2026-07-20T10:01:00Z' }),
    ];
    (element as any).commitments = new Map();
    (element as any).selectedMessageId = 'm2';
    document.body.appendChild(element);
    await (element as any).updateComplete;

    const nodes = element.shadowRoot!.querySelectorAll('.flow-node');
    expect(nodes.length).toBe(2);
  });

  it('shows duration between nodes', async () => {
    element = document.createElement('channel-correlation-panel') as any;
    (element as any).messages = [
      makeMsg({ id: 'm1', correlationId: 'corr-1', createdAt: '2026-07-20T10:00:00Z' }),
      makeMsg({ id: 'm2', correlationId: 'corr-1', createdAt: '2026-07-20T10:05:00Z' }),
    ];
    (element as any).commitments = new Map();
    (element as any).selectedMessageId = 'm1';
    document.body.appendChild(element);
    await (element as any).updateComplete;

    const duration = element.shadowRoot!.querySelector('.flow-duration');
    expect(duration?.textContent).toContain('5m');
  });

  it('emits message-selected on node click', async () => {
    element = document.createElement('channel-correlation-panel') as any;
    const msg = makeMsg({ id: 'click-me', correlationId: 'corr-1' });
    (element as any).messages = [msg];
    (element as any).commitments = new Map();
    (element as any).selectedMessageId = 'click-me';
    document.body.appendChild(element);
    await (element as any).updateComplete;

    let emitted: any = null;
    element.addEventListener('pages-event', (e: Event) => { emitted = (e as CustomEvent).detail; });

    const node = element.shadowRoot!.querySelector('.flow-node') as HTMLElement;
    node.click();

    expect(emitted).not.toBeNull();
    expect(emitted.payload.message.id).toBe('click-me');
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `cd /Users/mdproctor/claude/casehub/blocks-ui/components/channel-activity && npx vitest run src/channel-correlation-panel.test.ts`
Expected: FAIL — `./channel-correlation-panel.js` does not exist

- [ ] **Step 3: Create channel-correlation-panel.ts**

Create `components/channel-activity/src/channel-correlation-panel.ts` — adapted from Claudony's `claudony-correlation-panel.ts` with:
- Tag: `channel-correlation-panel`, Class: `ChannelCorrelationPanelElement`
- Imports: `QhorusMessage`, `messageTypeCategory`, `commitmentStateCategory`, `isObligationCreating` from `./types.js`; `ChannelEventTopics` from `./events.js`; `CommitmentRecord` from `./commitment.js`; `emitPagesEvent` from `@casehubio/blocks-ui-core`
- All CSS fallbacks stripped
- HTMLElementTagNameMap declares `'channel-correlation-panel'`

The full file follows the same structure as the Claudony source (232 lines) with the transformations listed above applied throughout.

- [ ] **Step 4: Run tests**

Run: `cd /Users/mdproctor/claude/casehub/blocks-ui/components/channel-activity && npx vitest run src/channel-correlation-panel.test.ts`
Expected: ALL 5 PASS

- [ ] **Step 5: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/blocks-ui add components/channel-activity/src/channel-correlation-panel.ts components/channel-activity/src/channel-correlation-panel.test.ts
git -C /Users/mdproctor/claude/casehub/blocks-ui commit -m "feat(#180): add channel-correlation-panel component

Refs casehubio/claudony#180"
```

---

### Task 4: Channel artifact panel (blocks-ui)

**Files:**
- Create: `components/channel-activity/src/channel-artifact-panel.ts`
- Create: `components/channel-activity/src/channel-artifact-panel.test.ts`

**Interfaces:**
- Consumes: `ArtefactRef`, `ResolvedArtifact` from `./types.js`; `emitPagesEvent` from `@casehubio/blocks-ui-core` (not currently used in artifact panel but available)
- Produces: `<channel-artifact-panel>` element, `ChannelArtifactPanelElement` class

- [ ] **Step 1: Write channel-artifact-panel.test.ts**

Create `components/channel-activity/src/channel-artifact-panel.test.ts`:

```typescript
import { describe, it, expect, afterEach } from 'vitest';
import './channel-artifact-panel.js';
import type { ArtefactRef, ResolvedArtifact } from './types.js';

function makeRef(overrides: Partial<ArtefactRef> = {}): ArtefactRef {
  return { uri: 'doc://case-123/report.md', type: 'DOCUMENT', label: 'Case Report', ...overrides };
}

describe('channel-artifact-panel', () => {
  let element: HTMLElement;

  afterEach(() => { element?.remove(); });

  it('shows empty state when no artifact selected', async () => {
    element = document.createElement('channel-artifact-panel') as any;
    document.body.appendChild(element);
    await (element as any).updateComplete;

    expect(element.shadowRoot!.querySelector('.empty')?.textContent).toContain('Select a message');
  });

  it('renders artifact header with label and type badge', async () => {
    element = document.createElement('channel-artifact-panel') as any;
    (element as any).selectedArtefactRef = makeRef({ label: 'Risk Report', type: 'DOCUMENT' });
    document.body.appendChild(element);
    await (element as any).updateComplete;

    expect(element.shadowRoot!.querySelector('.artifact-label')?.textContent).toBe('Risk Report');
    expect(element.shadowRoot!.querySelector('.type-badge')?.textContent).toBe('DOCUMENT');
  });

  it('renders card view for CASE type artifacts', async () => {
    element = document.createElement('channel-artifact-panel') as any;
    (element as any).selectedArtefactRef = makeRef({ type: 'CASE', label: 'Case AML-4521', uri: 'case://aml-4521' });
    document.body.appendChild(element);
    await (element as any).updateComplete;

    const card = element.shadowRoot!.querySelector('.artifact-card');
    expect(card).not.toBeNull();
    expect(card?.querySelector('.card-label')?.textContent).toBe('Case AML-4521');
  });

  it('calls resolveArtifact callback and renders content', async () => {
    element = document.createElement('channel-artifact-panel') as any;
    const ref = makeRef({ type: 'CODE', label: 'main.ts' });
    (element as any).resolveArtifact = async (_r: ArtefactRef): Promise<ResolvedArtifact> =>
      ({ content: 'console.log("hello")', language: 'typescript' });
    (element as any).selectedArtefactRef = ref;
    document.body.appendChild(element);
    await (element as any).updateComplete;
    await new Promise(r => setTimeout(r, 10));
    await (element as any).updateComplete;

    expect(element.shadowRoot!.querySelector('.content-text')?.textContent).toBe('console.log("hello")');
  });

  it('maintains navigation history', async () => {
    element = document.createElement('channel-artifact-panel') as any;
    document.body.appendChild(element);

    (element as any).selectedArtefactRef = makeRef({ uri: 'doc://a', label: 'A' });
    await (element as any).updateComplete;

    (element as any).selectedArtefactRef = makeRef({ uri: 'doc://b', label: 'B' });
    await (element as any).updateComplete;

    const backBtn = element.shadowRoot!.querySelector('.nav-back') as HTMLButtonElement;
    expect(backBtn.disabled).toBe(false);

    backBtn.click();
    await (element as any).updateComplete;
    expect(element.shadowRoot!.querySelector('.artifact-label')?.textContent).toBe('A');
  });

  it('renders scope highlight when selectedText present', async () => {
    element = document.createElement('channel-artifact-panel') as any;
    (element as any).selectedArtefactRef = makeRef({
      scope: { startLine: 10, endLine: 15, selectedText: 'highlighted section' },
    });
    document.body.appendChild(element);
    await (element as any).updateComplete;

    const highlight = element.shadowRoot!.querySelector('.scope-highlight');
    expect(highlight?.textContent).toContain('highlighted section');
    expect(highlight?.textContent).toContain('10');
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `cd /Users/mdproctor/claude/casehub/blocks-ui/components/channel-activity && npx vitest run src/channel-artifact-panel.test.ts`
Expected: FAIL — `./channel-artifact-panel.js` does not exist

- [ ] **Step 3: Create channel-artifact-panel.ts**

Create `components/channel-activity/src/channel-artifact-panel.ts` — adapted from Claudony's `claudony-artifact-panel.ts` with:
- Tag: `channel-artifact-panel`, Class: `ChannelArtifactPanelElement`
- Imports: `ArtefactRef`, `ResolvedArtifact` from `./types.js`
- `resolveArtifact` callback typed as `(ref: ArtefactRef) => Promise<ResolvedArtifact>` instead of anonymous inline type
- All CSS fallbacks stripped
- HTMLElementTagNameMap declares `'channel-artifact-panel'`

The full file follows the same structure as the Claudony source (258 lines) with the transformations applied.

- [ ] **Step 4: Run tests**

Run: `cd /Users/mdproctor/claude/casehub/blocks-ui/components/channel-activity && npx vitest run src/channel-artifact-panel.test.ts`
Expected: ALL 6 PASS

- [ ] **Step 5: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/blocks-ui add components/channel-activity/src/channel-artifact-panel.ts components/channel-activity/src/channel-artifact-panel.test.ts
git -C /Users/mdproctor/claude/casehub/blocks-ui commit -m "feat(#180): add channel-artifact-panel component

Refs casehubio/claudony#180"
```

---

### Task 5: Exports, showcase, version bump, build + publish (blocks-ui)

**Files:**
- Modify: `components/channel-activity/src/index.ts`
- Modify: `components/channel-activity/package.json` (version bump)
- Modify: `examples/src/pages/channel-activity-page.ts` (showcase)

**Interfaces:**
- Consumes: All three panel elements + commitment module from Tasks 1-4

- [ ] **Step 1: Update index.ts exports**

Add to `components/channel-activity/src/index.ts`:

```typescript
export * from './commitment.js';
export { ChannelTaskPanelElement } from './channel-task-panel.js';
export { ChannelCorrelationPanelElement } from './channel-correlation-panel.js';
export { ChannelArtifactPanelElement } from './channel-artifact-panel.js';
```

- [ ] **Step 2: Bump version**

In `components/channel-activity/package.json`, change `"version": "0.2.2"` → `"version": "0.3.0"`.

- [ ] **Step 3: Add showcase sections to channel-activity-page.ts**

Add new demo sections after the existing demos in `examples/src/pages/channel-activity-page.ts`. Import `CommitmentRecord` from `@casehubio/blocks-ui-channel-activity`. Add mock data using valid `MessageType` values (`COMMAND`, `STATUS`, `RESPONSE`, `DONE`, `FAILURE`, `DECLINE`, `HANDOFF`) and showcase all three panels:

**Task panel demo** — COMMAND messages with commitment records in all three groups (active, overdue, completed). Add a `setInterval` that transitions a commitment from OPEN → ACKNOWLEDGED → FULFILLED over 5-second intervals.

**Correlation panel demo** — Messages with `correlationId: 'corr-demo'` forming a command→status→done chain. Pre-select the first message via `@state()`.

**Artifact panel demo** — Messages with `artefactRefs` of types DOCUMENT, CODE, CASE, EXTERNAL. Wire `resolveArtifact` to return mock content. Pre-select the first artifact.

- [ ] **Step 4: Run all tests**

Run: `cd /Users/mdproctor/claude/casehub/blocks-ui/components/channel-activity && npx vitest run`
Expected: ALL PASS (existing + 18 new tests across commitment, task, correlation, artifact)

- [ ] **Step 5: Build**

Run: `cd /Users/mdproctor/claude/casehub/blocks-ui/components/channel-activity && npx tsc -p tsconfig.build.json`
Expected: Clean build, no errors

- [ ] **Step 6: Publish**

Run: `cd /Users/mdproctor/claude/casehub/blocks-ui/components/channel-activity && npm publish`
Expected: Published `@casehubio/blocks-ui-channel-activity@0.3.0`

- [ ] **Step 7: Commit and push**

```bash
git -C /Users/mdproctor/claude/casehub/blocks-ui add components/channel-activity/src/index.ts components/channel-activity/package.json examples/src/pages/channel-activity-page.ts
git -C /Users/mdproctor/claude/casehub/blocks-ui commit -m "feat(#180): export panels, showcase demos, bump to 0.3.0

Refs casehubio/claudony#180"
git -C /Users/mdproctor/claude/casehub/blocks-ui push origin main
```

---

### Task 6: Claudony migration

**Files:**
- Delete: `app/src/main/webui/src/components/claudony-task-panel.ts`
- Delete: `app/src/main/webui/src/components/claudony-correlation-panel.ts`
- Delete: `app/src/main/webui/src/components/claudony-artifact-panel.ts`
- Modify: `app/src/main/webui/src/util/channel-adapter.ts` (remove commitment types/functions)
- Modify: `app/src/main/webui/src/util/channel-adapter.test.ts` (remove commitment tests)
- Modify: `app/src/main/webui/src/components/claudony-workbench.ts` (update imports + tag names)
- Modify: `app/src/main/webui/package.json` (bump dependency)
- Modify: `CLAUDE.md` (update project structure)

**Interfaces:**
- Consumes: Published `@casehubio/blocks-ui-channel-activity@0.3.0`

- [ ] **Step 1: Bump dependency and install**

In `app/src/main/webui/package.json`, change `"@casehubio/blocks-ui-channel-activity": "^0.2.2"` → `"@casehubio/blocks-ui-channel-activity": "^0.3.0"`.

Run: `cd /Users/mdproctor/claude/casehub/claudony/app/src/main/webui && GITHUB_TOKEN=$(gh auth token) npm install`

- [ ] **Step 2: Delete the three panel files**

```bash
rm /Users/mdproctor/claude/casehub/claudony/app/src/main/webui/src/components/claudony-task-panel.ts
rm /Users/mdproctor/claude/casehub/claudony/app/src/main/webui/src/components/claudony-correlation-panel.ts
rm /Users/mdproctor/claude/casehub/claudony/app/src/main/webui/src/components/claudony-artifact-panel.ts
```

- [ ] **Step 3: Update channel-adapter.ts**

Remove from `channel-adapter.ts`:
- `CommitmentRecord` interface (lines 83-89)
- `RawCommitment` interface (lines 91-101)
- `toCommitmentRecord` function (lines 103-117)
- `toCommitmentMap` function (lines 119-127)

Keep: `TimelineEntry`, `ChannelInfo`, `resolveActorType`, `toQhorusMessage`, `toQhorusChannel`, `formatEventContent`.

Remove `CommitmentState` from the import on line 1 (no longer used in this file).

- [ ] **Step 4: Update channel-adapter.test.ts**

Remove the import of `toCommitmentRecord, toCommitmentMap` from line 2.

Remove the entire `describe('CommitmentRecord mapping', ...)` block (lines 179-216).

- [ ] **Step 5: Update claudony-workbench.ts**

Change line 7 from:
```typescript
import { toQhorusMessage, toQhorusChannel, toCommitmentMap, type TimelineEntry, type ChannelInfo, type CommitmentRecord } from '../util/channel-adapter.js';
```
to:
```typescript
import { toQhorusMessage, toQhorusChannel, type TimelineEntry, type ChannelInfo } from '../util/channel-adapter.js';
import { toCommitmentMap, type CommitmentRecord } from '@casehubio/blocks-ui-channel-activity';
```

Replace lines 9-11 (side-effect imports of the three panel files):
```typescript
import './claudony-task-panel.js';
import './claudony-correlation-panel.js';
import './claudony-artifact-panel.js';
```
with:
```typescript
import '@casehubio/blocks-ui-channel-activity';
```

In the template (render method), replace all three tag names:
- `claudony-task-panel` → `channel-task-panel`
- `claudony-correlation-panel` → `channel-correlation-panel`
- `claudony-artifact-panel` → `channel-artifact-panel`

- [ ] **Step 6: Run Claudony vitest**

Run: `cd /Users/mdproctor/claude/casehub/claudony && GITHUB_TOKEN=$(gh auth token) npm --prefix app/src/main/webui test`
Expected: ALL PASS (fewer tests — commitment tests removed)

- [ ] **Step 7: Run Claudony Java tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl claudony-app`
Expected: ALL PASS

- [ ] **Step 8: Update CLAUDE.md**

In the `claudony-app/src/main/webui/` project structure section, remove the three panel entries:
```
│       ├── claudony-task-panel.ts     — commitment tracking panel ...
│       ├── claudony-correlation-panel.ts — conversation chain visualization ...
│       ├── claudony-artifact-panel.ts — artifact reference viewer ...
```

Replace with a note that they moved:
```
│       │                                  Task, correlation, and artifact panels promoted to
│       │                                  @casehubio/blocks-ui-channel-activity (channel-task-panel,
│       │                                  channel-correlation-panel, channel-artifact-panel).
```

- [ ] **Step 9: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/claudony add app/src/main/webui/src/components/claudony-task-panel.ts app/src/main/webui/src/components/claudony-correlation-panel.ts app/src/main/webui/src/components/claudony-artifact-panel.ts app/src/main/webui/src/util/channel-adapter.ts app/src/main/webui/src/util/channel-adapter.test.ts app/src/main/webui/src/components/claudony-workbench.ts app/src/main/webui/package.json app/src/main/webui/package-lock.json CLAUDE.md
git -C /Users/mdproctor/claude/casehub/claudony commit -m "refactor(#180): consume panels from @casehubio/blocks-ui-channel-activity

Removes claudony-task-panel, claudony-correlation-panel,
claudony-artifact-panel. Now imported from blocks-ui as
channel-task-panel, channel-correlation-panel, channel-artifact-panel.

CommitmentRecord and conversion functions also moved to blocks-ui.

Closes #180"
```
