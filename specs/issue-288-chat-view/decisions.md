# Design Decisions — #288 Chat View

## D1: Solve parity structurally, not by copying

**Choice:** Extract a reusable `<conversation-workbench>` into casehub-mosaic, then wire it in claudony. Chat-app migrates separately.
**Alternatives:**
- Copy chat-app's workbench into claudony — faster but perpetuates dual maintenance
- Chat route only in claudony — doesn't solve the parity problem
**Rationale:** Both claudony and chat-app compose the same mosaic controllers and components. The composition layer is the only missing piece in mosaic. Adding it means new features land once.
**Trade-offs:** Requires a mosaic release before claudony can consume it. Cross-repo coordination.
**Sources:** `chat-app/src/main/webui/src/workbench/qhorus-workbench.ts`, `claudony/app/src/main/webui/src/components/claudony-workbench.ts`
**Exploration:** quick
**Status:** captured

## D2: Config surface — callback-based, not path-based

**Choice:** The workbench takes callback functions for app-specific operations: `sendMessage`, `markRead`, space CRUD, corrections, retractions. Plus configuration for: auth fetch function, push WebSocket URL, identity/currentActorId, channel filter predicate.
**Alternatives:**
- RestBase + path template — less flexible, can't handle structural API differences
- Standardise REST shape across apps — ideal but much more work, separate effort
**Rationale:** Chat-app posts to `/api/chat/{channelId}/messages`, claudony posts to `/api/mesh/channels/{name}/messages`. These are structurally different. A callback lets each app implement delivery however it needs without the workbench knowing about REST paths.
**Trade-offs:** More props to configure. Apps must provide callbacks for every REST operation they want to support.
**Sources:** `chat-app/src/main/webui/src/auth.ts`, `claudony/app/src/main/webui/src/util/auth.ts`
**Exploration:** quick
**Status:** captured

## D3: Channel filtering as a config prop

**Choice:** Workbench takes an optional channel filter (predicate function or namespace prefix) to scope which channels appear.
**Alternatives:**
- No filtering — always show all channels (too broad for case-scoped views)
- Filter only in channel-nav — doesn't affect controller state
**Rationale:** Claudony has namespace conventions (`case-{uuid}/`, `life/`, `team/`). A case-bound view should only show that case's channels. Chat-app shows everything.
**Trade-offs:** Filter must be applied at the controller level so unread counts, presence, etc. are also scoped.
**Sources:** `docs/CHANNELS.md` (namespace conventions)
**Exploration:** quick
**Status:** captured

## D4: Space management — built-in, opt-in via callbacks

**Choice:** Workbench renders space management UI (create/rename/delete space, move channel to space) when space callbacks are provided. If callbacks are absent, space management is hidden.
**Alternatives:**
- Always built-in — forces every consumer to support spaces
- Separate component — more composition work for apps that want spaces
**Rationale:** Feature detection via config. Claudony may not support spaces initially; chat-app does. The UI should adapt to what's available.
**Trade-offs:** Conditional rendering adds complexity to the workbench template.
**Depends on:** D2 (callback-based config)
**Sources:** `chat-app/src/main/webui/src/workbench/qhorus-workbench.ts` (space event handling)
**Exploration:** quick
**Status:** captured

## D5: Layout persistence via optional store prop

**Choice:** Workbench takes an optional `layoutStore` prop. If provided, persists dock state, panel widths, and density. If absent, uses defaults and nothing persists.
**Alternatives:**
- Built-in localStorage — simpler but less flexible (key collisions, no custom backends)
- No persistence — apps handle externally via change events (more work for consumers)
**Rationale:** Apps control the storage key and backend. Chat-app uses `createLocalLayoutStore`; claudony may use a different mechanism or skip persistence entirely.
**Trade-offs:** Apps must create and pass a store if they want persistence.
**Sources:** `chat-app/src/main/webui/src/workbench/qhorus-workbench.ts` (layoutStore usage)
**Exploration:** quick
**Status:** captured

## D6: Claudony navigation — Chat as a fleet home tab

**Choice:** Chat view appears as a new tab in the fleet home alongside Sessions, Cases, Inbox, Fleet, Mesh.
**Alternatives:**
- Separate route `/app/chat` — independent page, better for deep-linking
- Both tab and standalone route — more flexible but two entry points to maintain
**Rationale:** Consistent with claudony's existing navigation pattern. Users switch between tabs without leaving the page.
**Trade-offs:** Full-viewport conversation within a tab may feel constrained. Deep-linking to a specific channel requires URL parameter support within the tab.
**Sources:** `claudony/app/src/main/webui/src/app.ts` (fleet home tabs)
**Exploration:** quick
**Status:** captured
