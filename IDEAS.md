# Idea Log

Undecided possibilities — things worth remembering but not yet decided.
Promote to an ADR when ready to decide; discard when no longer relevant.

---

## 2026-04-18 — Speech acts as a framework for Qhorus message types

**Status:** moved

This is a Qhorus concern, not a Claudony concern. `MessageType` is defined in Qhorus and used by all agents — richer semantic types at the infrastructure layer benefit every consumer, not just Claudony's human interjection panel.

**Moved to:** `~/claude/casehub/qhorus/IDEAS.md` on 2026-04-23.

---

## 2026-04-22 — Unified UI across CaseHub, Qhorus, and Claudony

**Status:** parked

The three-panel Claudony dashboard (case graph / terminal / side panel) is Claudony-specific today. As CaseHub and Qhorus mature, there will likely be a need for a unified UI that spans all three — surfacing case lineage, channel conversations, and session terminals in one place without Claudony being the mandatory host.

Open questions: Does the UI live in Claudony, in CaseHub, or as a standalone app? Does it require a backend aggregator? How does it handle fleet-distributed state (sessions on multiple nodes, channels on a shared DB)?

Not worth designing now — the individual UIs don't exist yet at the scale where unification is the bottleneck. Revisit when CaseHub Phase B dashboard is built and the seams between the three become visible in practice.

**Context:** ADR 0005 defers this explicitly — UI unification is out of scope for the CaseHub integration module decision.

---

# Idea Log
This is a living log of ideas, findings, and "things to return to" captured
during development. Not decisions (see docs/adr/), not design state
(see docs/design-snapshots/). Just possibilities worth remembering.

---

## [2026-04-03] iTerm2 integration is local-server-only — headless server breaks it
**Status:** Open
**Priority:** Important
**Context:** User may run Quarkus on a headless Mac Mini and access remotely
from MacBook + iPad. iTerm2 AppleScript runs on the server machine, so it
only works when the server is on the same Mac as iTerm2.

When server is remote (Mac Mini), iTerm2 integration should be hidden/disabled
in the dashboard. Dashboard detects this by checking if it's connecting to
localhost vs a remote host.

Future idea: a companion native Mac app or menu bar utility on the MacBook
that listens for "open in iTerm2" events from the remote server — would
restore iTerm2 integration even when the server is on Mac Mini.

Also surfaces: network binding must be configurable. Server must listen on
`0.0.0.0` (not just `localhost`) when deployed on Mac Mini. Should be a
config option, defaulting to localhost for safety.

Also surfaces: authentication becomes important when server is on a separate
networked machine. Currently undesigned — must address before Mac Mini deployment.

**Resolution needed before:** Implementing iTerm2 integration and network config.

---

## [2026-04-04] rename-session doesn't check tmux exit code
**Status:** Open
**Priority:** Worth Exploring
**Context:** Code review of SessionResource.rename() — if `tmux rename-session` fails (e.g. duplicate name), the registry is silently updated with the new name, causing split-brain between tmux state and registry state.
Fix: capture `p.waitFor()` return value and return 500 if non-zero. Consider also moving rename into TmuxService.

---

## [2026-04-04] ServerStartupTest hardcodes tmux prefix
**Status:** Open
**Priority:** Someday
**Context:** ServerStartupTest.java hardcodes `"claudony-"` as the expected prefix instead of reading from config. If `claudony.tmux-prefix` is changed in config, the test would silently pass vacuously (no sessions would match either the hardcoded or configured prefix). Should inject ClaudonyConfig and use config.tmuxPrefix().

---

## [2026-04-04] GraalVM not installed — native image build untested
**Status:** Implemented
**Priority:** Important
**Context:** Task 8 created reflect-config.json but couldn't verify native build. GraalVM IS installed — graalvm-25.jdk at /Library/Java/JavaVirtualMachines/ and via SDKMAN at ~/.sdkman/candidates/java/25.0.2-graalce/. native-image just wasn't on PATH. Native build completed successfully — see git log.

---

## [2026-04-03] pty4j incompatible with Quarkus native image
**Status:** Open
**Priority:** Important
**Context:** We chose pty4j as the Java PTY library for managing terminal sessions,
then committed to Quarkus native image compilation.

pty4j (JetBrains) uses JNI extensively and bundles native `.dylib`/`.so` libraries.
GraalVM native image has limited and complex support for JNI and dynamic native
library loading — pty4j will likely not work out of the box in native mode.

Possible alternatives to investigate:
1. **ProcessBuilder only** — spawn `tmux attach-session -t <name>` directly.
   tmux owns the PTY; Quarkus just pipes stdin/stdout over the WebSocket.
   Simpler, native-image safe, but less direct PTY control.
2. **`script -q /dev/null tmux attach-session -t <name>`** — forces PTY
   allocation on macOS without needing a Java PTY library.
3. **GraalVM JNI config** — attempt to configure native image reflection/JNI
   metadata for pty4j. Complex and fragile.
4. **Run Quarkus in JVM mode** — drop the native image requirement. Loses
   fast startup / low memory benefit.

Likely answer: Option 1 (ProcessBuilder + tmux attach) is the right call —
it's simpler, debuggable, native-safe, and tmux handling the PTY is actually
more correct given tmux is our session source of truth anyway.

**Resolution needed before:** Starting WebSocket terminal implementation.

---

## [2026-04-03] Webmux ideas worth revisiting as future enhancements
**Status:** Open
**Priority:** Someday
**Context:** We reviewed Webmux during tool research and decided not to use it,
but identified several patterns worth stealing later.

- **GitHub PR/CI status per session** — show PR status and CI badges inline
  in the session dashboard, no context-switching to GitHub needed
- **Docker sandbox per session** — run Claude sessions in isolated containers
  for untrusted or experimental work
- **Lifecycle hooks** — run scripts on session create/delete events
  (e.g. auto-setup env vars, notify a webhook, log session activity)
- **Auto-naming via AI** — call Claude API to suggest a session name based
  on the initial prompt or working directory
- **Service health badges** — poll configured ports (dev servers) and show
  live health indicators per session in the dashboard

**Resolution needed before:** Each is independent — revisit individually
when the core session management UI is working.

---

## [2026-04-03] iPad keyboard bar needed for full Claude Code experience
**Status:** Open
**Priority:** Important
**Context:** Discussing xterm.js on iPad — the software keyboard lacks Escape,
Ctrl+C, Tab, backtick, pipe, and tilde, which are all heavily used in Claude Code.

Every terminal-on-iPad approach hits this problem. The solution is a custom
key bar rendered above the xterm.js terminal in the web UI, providing one-tap
access to these keys.

Keys needed at minimum: `Escape`, `Ctrl+C`, `Ctrl+D`, `Tab`, `` ` ``, `|`, `~`, `↑`, `↓`
Bonus: configurable shortcuts (e.g. one-tap for common Claude Code commands)

This is not optional for a good iPad Claude Code experience — it is required.

**Resolution needed before:** Frontend implementation of the session terminal view.

---

## [2026-04-03] MCP transport: HTTP/SSE chosen but stdio worth understanding
**Status:** Open
**Priority:** Someday
**Context:** We chose HTTP/SSE for the MCP server since Quarkus is always running.
Most existing tmux MCP tools (conductor-mcp, tmux-claude-mcp-server) use stdio.

If we ever want the controller Claude to use those existing MCP servers alongside
ours, or if a user wants a different Claude client that only supports stdio,
we'd need to add stdio transport support. Not needed now, but worth noting
the gap.

**Resolution needed before:** Only if a specific need arises.

---

## [2026-04-03] tmux -CC integration mode for local iTerm2
**Status:** Open
**Priority:** Worth Exploring
**Context:** Decided that local iTerm2 access is optional — users can open an
iTerm2 window that attaches to a tmux session via tmux -CC (iTerm2 native
integration mode), preserving ~95% of iTerm2 features.

The MCP tool for "open in iTerm2" needs to use `tmux -CC` not plain
`tmux attach` to get the iTerm2 integration benefits. The AppleScript that
opens a new iTerm2 window should run:
  `tmux -CC attach-session -t <session-name>`
not:
  `tmux attach-session -t <session-name>`

Need to verify this works correctly in practice and document the AppleScript.

**Resolution needed before:** Implementing the iTerm2 AppleScript integration.
