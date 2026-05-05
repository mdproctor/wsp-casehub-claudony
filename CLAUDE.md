# claudony Workspace

**Project repo:** /Users/mdproctor/claude/casehub/claudony
**Workspace type:** public

## Session Start

Run `add-dir /Users/mdproctor/claude/casehub/claudony` before any other work.

## Artifact Locations

| Skill | Writes to |
|-------|-----------|
| brainstorming (specs) | `specs/` |
| writing-plans (plans) | `plans/` |
| handover | `HANDOFF.md` |
| idea-log | `IDEAS.md` |
| design-snapshot | `snapshots/` |
| java-update-design / update-primary-doc | `design/JOURNAL.md` (created by `epic`) |
| adr | `adr/` |
| write-blog | `blog/` |

## Structure

- `HANDOFF.md` — session handover (single file, overwritten each session)
- `IDEAS.md` — idea log (single file)
- `specs/` — brainstorming / design specs (superpowers output)
- `plans/` — implementation plans (superpowers output)
- `snapshots/` — design snapshots with INDEX.md (auto-pruned, max 10)
- `adr/` — architecture decision records with INDEX.md
- `blog/` — project diary entries with INDEX.md
- `design/` — epic journal (created by `epic` at branch start)

## Rules

- All methodology artifacts go here, not in the project repo
- Promotion to project repo is always explicit — never automatic
- Workspace branches mirror project branches — switch both together

## Routing

| Artifact   | Destination | Notes |
|------------|-------------|-------|
| adr        | workspace   | |
| blog       | workspace   | |
| design     | workspace   | |
| snapshots  | workspace   | |
| specs      | workspace   | |
| handover   | workspace   | |

---

# Claudony — Claude Code Project Guide

## Platform Context

This repo is one component of the casehubio multi-repo platform. **Before implementing anything — any feature, SPI, data model, or abstraction — run the Platform Coherence Protocol.**

The protocol asks: Does this already exist elsewhere? Is this the right repo for it? Does this create a consolidation opportunity? Is this consistent with how the platform handles the same concern in other repos?

**Platform architecture (fetch before any implementation decision):**
```
https://raw.githubusercontent.com/casehubio/parent/main/docs/PLATFORM.md
```

**This repo's deep-dive:**
```
https://raw.githubusercontent.com/casehubio/parent/main/docs/repos/claudony.md
```

**Other repo deep-dives** (fetch the relevant ones when your implementation touches their domain):
- casehub-ledger: `https://raw.githubusercontent.com/casehubio/parent/main/docs/repos/casehub-ledger.md`
- casehub-work: `https://raw.githubusercontent.com/casehubio/parent/main/docs/repos/casehub-work.md`
- casehub-qhorus: `https://raw.githubusercontent.com/casehubio/parent/main/docs/repos/casehub-qhorus.md`
- casehub-engine: `https://raw.githubusercontent.com/casehubio/parent/main/docs/repos/casehub-engine.md`
- casehub-connectors: `https://raw.githubusercontent.com/casehubio/parent/main/docs/repos/casehub-connectors.md`

---

## Project Type

**Type:** java

**Stack:** Java 21 (on Java 26 JVM), Quarkus 3.32.2, GraalVM 25 (native image), tmux, xterm.js

---

## What This Project Is

Claudony lets you run Claude Code CLI sessions on one machine (MacBook or headless Mac Mini) and access them from any device via a browser or PWA. A "controller" Claude instance manages sessions via MCP. Sessions persist independently — closing a browser tab or iTerm2 window never kills a session.

Two Quarkus modes from the same binary:
- **Server** — owns tmux sessions, WebSocket terminal streaming, web dashboard, REST API
- **Agent** — local MCP endpoint for controller Claude, iTerm2 integration, clipboard detection

---

## Build and Test

```bash
# Run all tests (all 3 modules)
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test

# Run only claudony-casehub integration tests
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl claudony-casehub

# Run specific test (searches all modules)
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -Dtest=ClassName

# Run real Claude E2E tests (requires claude CLI authenticated)
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -Pe2e

# Install Chromium for browser E2E tests (one-time per machine; uses local Maven repo)
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn dependency:copy-dependencies -DincludeGroupIds=com.microsoft.playwright -DoutputDirectory=/tmp/pw && \
  java -cp "/tmp/pw/*" com.microsoft.playwright.CLI install chromium

# Run browser E2E tests (requires Chromium installed above)
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -Pe2e -Dtest=PlaywrightSetupE2ETest,DashboardE2ETest,TerminalPageE2ETest

# Run with visible browser (local debugging)
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -Pe2e -Dplaywright.headless=false -Dtest=DashboardE2ETest

# JVM build (app module only — runnable jar)
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn package -DskipTests -pl claudony-app --also-make

# Native image build (requires GraalVM)
JAVA_HOME=/Library/Java/JavaVirtualMachines/graalvm-25.jdk/Contents/Home \
  mvn package -Pnative -DskipTests -pl claudony-app --also-make
```

**Use `mvn` not `./mvnw`** — the maven wrapper is broken on this machine.

---

## Running in Dev Mode

```bash
# Start server (dev mode, with hot reload)
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn quarkus:dev -Dclaudony.mode=server

# ⚠️ IMPORTANT: Hot reload breaks WebSocket endpoint registration in Quarkus dev mode.
# After ANY Java commit that triggers a reload, do a full server restart.
# See docs/BUGS-AND-ODDITIES.md entry #1 for details.

# Start JVM jar (IMPORTANT: -D flags must come BEFORE -jar, not after)
# Prod: session encryption key is auto-generated on first run and persisted to
# ~/.claudony/encryption-key — no env var needed. Set the env var only if you want
# to manage the key yourself (e.g. from a secrets manager); it takes precedence.
QUARKUS_HTTP_AUTH_SESSION_ENCRYPTION_KEY=<your-secret-32-chars> \
JAVA_HOME=$(/usr/libexec/java_home -v 26) java \
  -Dclaudony.mode=server -Dclaudony.bind=0.0.0.0 \
  -jar target/quarkus-app/quarkus-run.jar

JAVA_HOME=$(/usr/libexec/java_home -v 26) java \
  -Dclaudony.mode=agent -Dclaudony.port=7778 \
  -jar target/quarkus-app/quarkus-run.jar

# Start native binary
./target/claudony-1.0.0-SNAPSHOT-runner                     # server mode (default)
./target/claudony-1.0.0-SNAPSHOT-runner -Dclaudony.mode=agent -Dquarkus.http.port=7778

# Build Docker image (requires jar built first)
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn package -DskipTests
docker build -t claudony:latest .

# Run two-node fleet with docker compose
export CLAUDONY_FLEET_KEY=$(openssl rand -base64 32)
docker compose up
# Node A: http://localhost:7777  Node B: http://localhost:7778
```

**Default ports:** Server = 7777, Agent = 7778

---

## Key URLs (dev mode)

- Dashboard: `http://localhost:7777/app/`
- Health: `http://localhost:7777/q/health`
- Sessions API: `http://localhost:7777/api/sessions`
- MCP endpoint (Agent): `http://localhost:7778/mcp`
- WebSocket terminal: `ws://localhost:7777/ws/{session-id}`
- Register passkey: `http://localhost:7777/auth/register` (first run, or with invite token)
- Login: `http://localhost:7777/auth/login`
- Dev quick login: `http://localhost:7777/auth/dev-login` (POST — dev mode only, sets auth cookie)

---

## Java and GraalVM on This Machine

```bash
# Java 26 (Oracle, system default) — use for dev and tests
JAVA_HOME=$(/usr/libexec/java_home -v 26)

# GraalVM 25 — use for native image builds only
JAVA_HOME=/Library/Java/JavaVirtualMachines/graalvm-25.jdk/Contents/Home

# GraalVM is also at: ~/.sdkman/candidates/java/25.0.2-graalce
# native-image is NOT on PATH by default — must set JAVA_HOME explicitly
```

Compiler target is `release=21` (Java 21 API surface) but we compile on Java 26.
Virtual threads (`Thread.ofVirtual()`) work fine on Java 26 with release=21.

---

## Project Structure

3-module Maven project: `claudony-core` (shared services), `claudony-casehub` (CaseHub SPI implementations), `claudony-app` (Quarkus application).

```
claudony-core/src/main/java/dev/claudony/
├── config/ClaudonyConfig.java          — all config properties
└── server/
    ├── model/                          — Session (id, name, workingDir, command, status, createdAt,
    │                                       lastActive, expiryPolicy, caseId, roleName),
    │                                       SessionStatus, SessionExpiredEvent
    ├── TmuxService.java                — ProcessBuilder wrappers for tmux commands
    ├── SessionRegistry.java            — in-memory ConcurrentHashMap session store;
    │                                       findByCaseId(caseId) returns workers ordered by createdAt
    ├── WorkerCaseLifecycleEvent.java   — CDI event bridging casehub→app (avoids circular dep)
    └── expiry/                         — ExpiryPolicy SPI + implementations + scheduler

claudony-casehub/src/main/java/dev/claudony/casehub/
├── CaseHubConfig.java                  — @ConfigMapping for claudony.casehub.* properties
├── WorkerCommandResolver.java          — capability→command lookup with default fallback
├── CaseLineageQuery.java               — interface for prior worker queries (default: empty stub)
├── EmptyCaseLineageQuery.java          — @DefaultBean no-op impl (swap for JPA impl when casehub DB configured)
├── ClaudonyWorkerProvisioner.java      — WorkerProvisioner SPI: creates tmux sessions
├── ClaudonyCaseChannelProvider.java    — CaseChannelProvider SPI: Qhorus-backed channels
├── ClaudonyWorkerContextProvider.java  — WorkerContextProvider SPI: lineage + channel context + systemPrompt
├── ClaudonyWorkerStatusListener.java   — WorkerStatusListener SPI: lifecycle → SessionRegistry
├── WorkerSessionMapping.java           — role↔session bridge: caseId:role→sessionId + role→sessionId fallback
├── JpaCaseLineageQuery.java            — @Alternative @Priority(1): queries case_ledger_entry via qhorus PU
├── CaseChannelLayout.java              — SPI: controls which channels open per case; ChannelSpec record
├── NormativeChannelLayout.java         — default: work/observe/oversight channels (APPEND semantic)
├── SimpleLayout.java                   — 2-channel variant: work/observe only (no oversight)
├── MeshParticipationStrategy.java      — SPI: controls mesh engagement; MeshParticipation enum (ACTIVE/REACTIVE/SILENT)
├── ActiveParticipationStrategy.java    — default: register + STATUS + periodic check_messages
├── ReactiveParticipationStrategy.java  — engage only when directly addressed
├── SilentParticipationStrategy.java    — no mesh participation
├── MeshSystemPromptTemplate.java       — package-private: generates ACTIVE/REACTIVE/SILENT prompt from channels + prior workers
└── ClaudonyLedgerEventCapture.java     — @ObservesAsync CaseLifecycleEvent → writes CaseLedgerEntry via @LedgerPersistenceUnit EM

claudony-app/src/main/java/dev/claudony/
├── server/
│   ├── SessionResource.java            — REST API /api/sessions (+ GET /{id}/lineage → CaseLineageQuery;
│   │                                       GET /api/sessions/{id}/case-events SSE stream)
│   ├── TerminalWebSocket.java          — WebSocket /ws/{id}, pipe-pane + FIFO streaming
│   ├── ServerStartup.java              — startup health checks, directory creation, tmux bootstrap
│   ├── CaseWorkerUpdateStrategy.java   — SPI: events-only | hybrid | registry-hooks
│   ├── CaseEventBroadcaster.java       — @ApplicationScoped SSE fan-out; observes WorkerCaseLifecycleEvent
│   ├── strategy/
│   │   ├── EventsOnlyStrategy.java     — emits on lifecycle events only
│   │   ├── HybridStrategy.java         — events + configurable heartbeat (default 30s)
│   │   └── RegistryHooksStrategy.java  — fires on any SessionRegistry mutation
│   ├── fleet/
│   │   ├── PeerRegistry.java           — authoritative peer list, circuit breaker, atomic peers.json persistence
│   │   ├── PeerHealthScheduler.java    — @Scheduled health check loop, per-peer virtual thread
│   │   ├── PeerResource.java           — REST /api/peers (CRUD + /{id}/sessions + /{id}/ping + /generate-fleet-key)
│   │   ├── PeerClient.java             — @RegisterRestClient for peer /api/sessions calls
│   │   ├── StaticConfigDiscovery.java  — loads claudony.peers at startup
│   │   ├── ManualRegistrationDiscovery.java — REST-triggered peer management, persisted to peers.json
│   │   └── MdnsDiscovery.java          — mDNS advertise/discover (scaffold; full impl follow-on)
│   └── auth/
│       ├── ApiKeyService.java          — key resolution (config → file → generate), first-run banner
│       ├── ApiKeyAuthMechanism.java    — X-Api-Key header auth (Agent→Server) + dev cookie
│       ├── AuthResource.java           — /auth/register, /auth/login, /auth/dev-login
│       ├── CredentialStore.java        — WebAuthn credential persistence (~/.claudony/credentials.json)
│       └── InviteService.java          — invite token generation and validation
└── agent/
    ├── ServerClient.java               — typed REST client to Server
    ├── ApiKeyClientFilter.java         — injects X-Api-Key on all ServerClient calls
    ├── McpServer.java                  — JSON-RPC POST /mcp (8 tools)
    ├── ClipboardChecker.java           — tmux clipboard detection/fix
    ├── AgentStartup.java               — Agent-mode startup checks
    └── terminal/
        ├── TerminalAdapter.java        — pluggable terminal interface
        ├── ITerm2Adapter.java          — macOS AppleScript + tmux -CC
        └── TerminalAdapterFactory.java — auto-detection

claudony-app/src/main/resources/META-INF/resources/  — static frontend served by Quarkus
├── manifest.json + sw.js              — PWA
└── app/
    ├── index.html + dashboard.js      — session management dashboard
    ├── session.html + terminal.js     — xterm.js terminal view + iPad key bar;
    │                                    when session.caseId set: case context panel
    │                                    (role, status, elapsed, lineage, channel auto-select)
    └── style.css                      — shared dark theme
```

### CaseHub integration

Enabled via `claudony.casehub.enabled=true`. Add to `application.properties`:

```properties
claudony.casehub.enabled=true
claudony.casehub.workers.commands.default=claude
# claudony.casehub.workers.commands."code-reviewer"=claude --mcp http://localhost:7778/mcp
claudony.casehub.workers.default-working-dir=~/claudony-workspace
claudony.casehub.channel-layout=normative      # normative | simple
claudony.casehub.mesh-participation=active     # active | reactive | silent
```

`ClaudonyWorkerProvisioner` creates tmux sessions with prefix `claudony-worker-{uuid}`.
`ClaudonyCaseChannelProvider` creates Qhorus channels named `case-{caseId}/{purpose}`.
`ClaudonyWorkerContextProvider` queries `CaseLineageQuery` for prior workers. The default `EmptyCaseLineageQuery` returns empty — swap for a `CaseLedgerEntryRepository`-backed implementation when a casehub datasource is configured.
`ClaudonyWorkerStatusListener` fires CDI `WorkerStalledEvent` on stall.

---

## Architecture Notes

**tmux is the source of truth.** Sessions live in tmux independent of the Quarkus server. On server restart, `ServerStartup` bootstraps the registry from `tmux list-sessions` (sessions with `claudony-` prefix). Working dir will show as "unknown" for bootstrapped sessions — this is expected.

**Terminal streaming (no PTY).** `tmux attach-session` requires a real PTY which ProcessBuilder cannot provide. We use `tmux pipe-pane` instead: pane output → FIFO → Java virtual thread → WebSocket. Input goes via `tmux send-keys -t name -l "text"` (the `-l` flag is critical — literal mode).

**History replay on reconnect.** Uses `tmux capture-pane -e -p -S -100` (ANSI colours). Lines are stripped of trailing whitespace (tmux pads to pane width), blank lines are removed (grid artefacts), joined with `\r\n` between lines (not after last), one trailing space restored on the last line (the current prompt). Sent synchronously BEFORE starting pipe-pane to avoid race conditions.

**MCP transport: HTTP/SSE.** The Agent exposes `POST /mcp` as a JSON-RPC endpoint. Claude Code connects to it as an MCP server. The Agent proxies session commands to the Server via REST. GraalVM-native compatible — no stdio process needed.

---

## Known Issues and Quirks

See `docs/BUGS-AND-ODDITIES.md` for comprehensive details. Key ones:

1. **Hot-reload breaks WebSocket** — full restart required after Java changes in dev mode
2. **One blank line after prompt on connect** — cosmetic, pipe-pane initial flush, harmless
3. **TUI apps (Claude Code, vim) history replay imperfect** — terminal resize triggers correct redraw
4. **Native binary staleness** — rebuild after adding new endpoints
5. **GraalVM not on PATH** — must set JAVA_HOME for native-image

---

## Configuration Properties

```properties
claudony.mode=server|agent
claudony.port=7777
claudony.bind=localhost                  # use 0.0.0.0 for Mac Mini / remote access
claudony.server.url=http://localhost:7777
claudony.claude-command=claude
claudony.tmux-prefix=claudony-
claudony.terminal=auto                   # auto|iterm2|none
claudony.default-working-dir=~/claudony-workspace   # default dir for new sessions
claudony.credentials-file=~/.claudony/credentials.json
claudony.agent.api-key=                  # auto-generated on first server run; saved to ~/.claudony/api-key
claudony.fleet-key=                     # shared secret for peer-to-peer API calls; generate with POST /api/peers/generate-fleet-key
claudony.peers=                         # comma-separated peer URLs for static discovery (e.g. http://mac-mini:7777)
claudony.mdns-discovery=false           # enable mDNS auto-discovery on LAN (scaffold; full impl follow-on)
claudony.name=Claudony                  # instance name shown in fleet dashboard
claudony.case-worker-update=hybrid      # events-only | hybrid | registry-hooks
claudony.case-worker-heartbeat-ms=30000 # heartbeat interval for hybrid strategy
# Production — optional; auto-generated and persisted to ~/.claudony/encryption-key on first run.
# Set only if managing the key externally (secrets manager, etc.):
# QUARKUS_HTTP_AUTH_SESSION_ENCRYPTION_KEY=<secret, >16 chars>

# Qhorus persistence (named datasource, Flyway-managed schema)
quarkus.datasource.qhorus.db-kind=h2
quarkus.datasource.qhorus.jdbc.url=jdbc:h2:file:~/.claudony/qhorus;DB_CLOSE_ON_EXIT=FALSE;AUTO_SERVER=TRUE
quarkus.hibernate-orm.qhorus.datasource=qhorus
quarkus.hibernate-orm.qhorus.packages=io.casehub.qhorus.runtime,io.casehub.ledger.runtime.model,io.casehub.ledger.model
quarkus.flyway.qhorus.migrate-at-start=true
# In future: change jdbc.url to PostgreSQL connection string for multi-instance fleet
```

**Directory convention:** `~/.claudony/` holds config/credentials (hidden, system); `~/.claudony/qhorus` is the Qhorus H2 database (shared data for fleet); `~/claudony-workspace/` is the default session working directory (visible, user-facing). All are created on server startup.

---

## Test Count and Status

**474 tests passing** (as of 2026-05-05, all modules): 4 in `claudony-core` + 134 in `claudony-casehub` + 336 in `claudony-app`. Zero failures, zero errors.

**Test convention — self-referencing REST clients:** In `@QuarkusTest` with `quarkus.http.test-port=0`, any REST client that calls back to the same running app must override its URL in `src/test/resources/application.properties`:
```properties
%test.claudony.server.url=http://localhost:${quarkus.http.port}
```
Quarkus resolves `${quarkus.http.port}` to the actual assigned random port. Without this, the client silently connects to the default port (7777) and all such tests fail with `Connection refused`.

**Qhorus tool count:** `McpServerIntegrationTest.toolsList_includesQhorusTools` asserts exactly 59 tools (8 Claudony + 51 Qhorus) — update this assertion and the count here when Qhorus ships new tools; the count changes with each Qhorus release. `quarkus.mcp.server.tools.page-size=0` in `application.properties` disables the default 50-tool pagination cap; the long-term fix (separate endpoints) is tracked in #105.

**casehub-ledger local build:** `casehub-ledger:0.2-SNAPSHOT` is not published to GitHub Packages — build and install it from source when the local repo is stale:
```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn install -DskipTests -q -pl casehub-ledger -am \
  -f /Users/mdproctor/claude/casehub/engine/pom.xml
```

`claudony-casehub` tests:
- `WorkerCommandResolverTest` — capability-to-command resolution, default fallback
- `ClaudonyWorkerProvisionerTest` — tmux session creation, disabled guard, terminate robustness, caseId/roleName stamped
- `ClaudonyCaseChannelProviderTest` — Qhorus channel creation, send, list filtering
- `ClaudonyWorkerContextProviderTest` — lineage, channel, clean-start, missing caseId
- `ClaudonyWorkerStatusListenerTest` — ACTIVE/IDLE/FAULTED lifecycle, stall event
- `NormativeChannelLayoutTest`, `SimpleLayoutTest` — channel specs, semantics, and allowedTypes for each layout
- `MeshParticipationStrategyTest` — ACTIVE/REACTIVE/SILENT strategy, robustness, correctness
- `WorkerSessionMappingTest` — role↔session bridge lookups
- `WorkerLifecycleSequenceTest` — full SPI lifecycle + meshParticipation in context
- `MeshSystemPromptTemplateTest` — 18 unit tests: ACTIVE/REACTIVE full templates, SILENT omitted, channel names, prior workers, correctness
- `JpaCaseLineageQueryTest` — JPA lineage query against qhorus PU

`claudony-app` tests (in `claudony-app/`):
- `SmokeTest` — basic health endpoint
- `server/` — TmuxService (real tmux; includes `displayMessage` tests), SessionRegistry (+ findByCaseId, 3 tests), SessionResource (+ ?caseId= filter, 2 tests; caseId/roleName in response, 1 test; case-events SSE endpoint), SessionLineageResourceTest (4 tests: happy path, no caseId, 404, non-UUID caseId robustness), TerminalWebSocket, ServerStartup, SessionInputOutput, MeshResourceInterjectionTest, `model/SessionTest` (session model + touch()), CaseEventBroadcasterTest (5 tests: events-only, hybrid heartbeat, registry-hooks, SSE fan-out, EventSource close)
- `server/strategy/` — EventsOnlyStrategyTest (5 unit tests), HybridStrategyTest (4 unit tests)
- `server/auth/` — ApiKeyService, ApiKeyAuthMechanism, AuthResource, AuthRateLimiter (+ AuthRateLimiterHttpTest for HTTP-level), CredentialStore, InviteService, FleetKeyService, FleetKeyAuth
- `server/expiry/` — ExpiryPolicyRegistryTest, UserInteractionExpiryPolicyTest, TerminalOutputExpiryPolicyTest, StatusAwareExpiryPolicyTest, SessionIdleSchedulerTest
- `config/` — EncryptionKeyConfigSource (15 unit tests + 5 QuarkusTest integration), SessionTimeoutConfigTest (3 QuarkusTest integration)
- `server/fleet/` — PeerRegistryTest (unit), StaticConfigDiscoveryTest (unit), MdnsDiscoveryTest (unit), PeerResourceTest (QuarkusTest + proxy resize), SessionFederationTest (QuarkusTest), ProxyWebSocketTest (QuarkusTest)
- `agent/` — McpServer (mocked), McpServerIntegrationTest (real HTTP), ServerClient, ClipboardChecker, ITerm2Adapter, TerminalAdapterFactory, AgentStartup
- `casehub/` — MeshParticipationIntegrationTest (full Quarkus context, ACTIVE — default config), MeshParticipationSilentProfileTest (SILENT config profile), `SystemPromptIntegrationTest`, `SystemPromptSilentProfileTest` — Quarkus integration: systemPrompt present for ACTIVE, absent for SILENT; `CaseLineageQueryIntegrationTest` — JPA integration: lineage query against real H2 with camelCase event types; `CaseEngineRoundTripTest` — CDI event→ledger→lineage round-trip: fires CaseLifecycleEvent, verifies ClaudonyLedgerEventCapture writes and JpaCaseLineageQuery reads back WorkerSummary; `ClaudonyLedgerEventCaptureTest` — 6 tests: happy path fields, sequence increment per case, sequence independence, null guards (2), worker event type
- `frontend/` — StaticFilesTest (all static files + content), AppAuthProtectionTest (/app/* unauthenticated), ResizeEndpointTest
- `e2e/` — ClaudeE2ETest (real `claude` CLI), PlaywrightSetupE2ETest (4 browser infra), DashboardE2ETest (7 dashboard UI), TerminalPageE2ETest (2: structure + proxy resize URL), ChannelPanelE2ETest (8: toggle, dropdown, timeline, badges, human sender, post message, cursor polling, Ctrl+K), CaseWorkerPanelE2ETest (7: standalone placeholder, CaseHub auto-expand + worker list, click-to-switch, regression guard SSE, initial SSE snapshot, SSE push update, EventSource close), CaseContextPanelE2ETest (4: case header with role/status, no header for standalone, lineage toggle expand/collapse, channel auto-select) — all via `mvn test -Pe2e -Dtest=...`, skipped in default run

**Playwright `<option>` element visibility:** Playwright 1.52+ considers `<option>` elements inside a `<select>` as "hidden" (zero rendered dimensions). Use `waitFor(new Locator.WaitForOptions().setState(WaitForSelectorState.ATTACHED))` instead of the default (`visible`) when waiting for options to be added to a dropdown. Default `waitFor()` will timeout even when the option is in the DOM.

**Browser test hook convention:** JavaScript that should only run during Playwright tests is gated behind `window.__CLAUDONY_TEST_MODE__`. Tests set it via `page.addInitScript("window.__CLAUDONY_TEST_MODE__ = true;")` before navigation. Never expose test hooks unconditionally.

`ServerStartup.bootstrapRegistry()` is package-private to allow direct testing.
Auth tests use `@TestSecurity(user = "test", roles = "user")` to bypass auth in non-auth test classes.
Stateful `@ApplicationScoped` beans (e.g. `AuthRateLimiter`) expose `resetForTest()` / `setClockForTest()` package-private hooks; `@AfterEach` cleanup is required to prevent state bleeding across `@QuarkusTest` classes, which share one app instance per test run.
**Qhorus test cleanup:** For Qhorus data (channels, messages, etc.): inject `@Inject InMemoryChannelStore channelStore` and `@Inject InMemoryMessageStore messageStore` (provided by the `casehub-qhorus-testing` dependency), then call `clear()` on both in `@AfterEach`. This is cleaner than the earlier `UserTransaction` pattern and works with all InMemory store implementations. Example: `MeshResourceInterjectionTest`.
**casehub-testing CDI isolation:** Adding `casehub-testing` to the test classpath requires `quarkus.index-dependency.casehub-testing.*` in `application.properties`. When indexed, casehub-engine's no-op SPI beans (`NoOpWorkerProvisioner` etc.) are `@ApplicationScoped` and collide with Claudony's own SPI implementations — add `quarkus.arc.exclude-types` listing each no-op class individually (wildcard `io.casehub.engine.internal.worker.*` exclusion breaks internal engine beans). Upstream fix needed: casehub-engine should use `@DefaultBean` for no-ops.
`%test.quarkus.datasource.reactive=false` is required in `application.properties` when a transitive dependency pulls in `hibernate-reactive-panache` — without it, H2 tests fail to start entirely.
`src/test/resources/application.properties` sets `quarkus.http.test-port=0` — assigns a random port per test run to prevent "Port already bound: 8081" when `mvn test` is run in quick succession (a lingering Surefire JVM from the previous run can hold the port).
**Playwright E2E base URL:** All E2E test classes extend `PlaywrightBase`. `BASE_URL` is resolved in `@BeforeAll` via `ConfigProvider.getConfig().getValue("test.url", String.class)` — Quarkus sets `test.url` to the actual bound URL after server startup. Do **not** hardcode `"http://localhost:8081"` or use a `static final` field — it breaks with random ports.

---

## Design Document

`docs/DESIGN.md` is the living architectural overview — updated via `/update-design` (or automatically by `java-git-commit`). For point-in-time snapshots, see `docs/design-snapshots/`.

## Ecosystem Context

Claudony is the integration layer in a three-project Quarkus Native AI Agent Ecosystem:

- **CaseHub** (`~/claude/casehub/engine`) — orchestration/choreography engine; defines SPIs that Claudony implements (`~/claude/casehub-poc` is the retiring POC — do not use)
- **Qhorus** (`~/claude/casehub/qhorus`) — agent communication mesh; Claudony embeds it and provides the dashboard observation layer
- **Claudony** (this project) — wires everything together; implements CaseHub SPIs, embeds Qhorus, hosts the unified dashboard

The canonical ecosystem design document lives here in this repo. It is the master architectural blueprint for all three projects — covering project topology, SPI contracts, MCP tool surfaces, choreography/orchestration use cases, the unified observer dashboard, human-in-the-loop interjection, and the B→C→A build roadmap.

Load it when working on: CaseHub SPI implementations, Qhorus embedding, the unified MCP endpoint, the three-panel dashboard, or any cross-project architectural decisions:

@/Users/mdproctor/claude/casehub/claudony/docs/superpowers/specs/2026-04-13-quarkus-ai-ecosystem-design.md

---

## Project Blog

Entries live in `docs/blog/`. Written using the personal technical writing
style guide at `~/claude-workspace/writing-styles/blog-technical.md`
(set `PERSONAL_WRITING_STYLES_PATH=~/claude-workspace/writing-styles`).

---

## Landing Page

The public site lives at `https://mdproctor.github.io/claudony/` and is a Jekyll 4 site in `docs/`.

**Deployed automatically** via `.github/workflows/jekyll.yml` on every push to `main` — no manual step needed.

**Structure:**
- `docs/_posts/` — blog entries (Jekyll format, mirrors `docs/blog-archive/`)
- `docs/guide/index.md` — getting started guide
- `docs/_layouts/` — page templates
- `docs/assets/` — CSS and JS

**Local development:**
```bash
cd docs && bundle exec jekyll serve --baseurl ""
# → http://localhost:4000
```

**Ruby:** Use Homebrew Ruby (`/opt/homebrew/opt/ruby/bin/bundle`) — system Ruby 2.6 is too old for Jekyll 4.

---

## What's Not Done Yet

- Authentication — WebAuthn passkey + API key implemented; rate limiting and dev-login backdoor closed; session expiry implemented (`SessionIdleScheduler` + pluggable `ExpiryPolicy`); session cookies expire on browser close; server restarts no longer invalidate sessions since encryption key is now persistent
- GitHub PR/CI integration in dashboard (idea logged)
- Docker sandbox per session (idea logged)
- Windows Terminal or Linux terminal adapters beyond iTerm2 (interface is pluggable, no implementation)

## Work Tracking

**Issue tracking:** enabled
**GitHub repo:** casehubio/claudony
**Changelog:** GitHub Releases (run `gh release create --generate-notes` at milestones)

**Automatic behaviours (Claude follows these at all times in this project):**
- **Before implementation begins** — when the user says "implement", "start coding",
  "execute the plan", "let's build", or similar: check if an active issue or epic
  exists. If not, run issue-workflow Phase 1 to create one **before writing any code**.
- **Before writing any code** — check if an issue exists for what's about to be
  implemented. If not, draft one and assess epic placement (issue-workflow Phase 2)
  before starting. Also check if the work spans multiple concerns.
- **Before any commit** — run issue-workflow Phase 3 (via git-commit) to confirm
  issue linkage and check for split candidates. This is a fallback — the issue
  should already exist from before implementation began.
- **All commits should reference an issue** — `Refs #N` (ongoing) or `Closes #N` (done).
  If the user explicitly says to skip ("commit as is", "no issue"), ask once to confirm
  before proceeding — it must be a deliberate choice, not a default.
  **Exception:** housekeeping commits (consistency passes, doc fixes, dependency bumps, health-check fixes) may omit issue links when no issue naturally applies. These should still use conventional commit format: `docs:`, `chore:`, `fix(deps):`, etc.
  **Commit scope examples:** `fix(auth): ...`, `feat(mcp): ...`, `docs(design): ...`, `chore(deps): ...`, `test(e2e): ...`, `refactor(sessions): ...`

## Ecosystem Conventions

All casehubio projects align on these conventions:

**Publishing strategy:** All casehub components (casehub-ledger, Qhorus, casehub-engine, Claudony, etc.) are sub-projects of the casehubio GitHub org. They are **not** submitted to Quarkiverse. Existing `io.quarkiverse.*` package names in dependencies are legacy naming — new casehub components use `io.casehub.*` conventions. Do not propose Quarkiverse publication for any casehub component.

**Quarkus version:** All projects use `3.32.2`. When bumping, bump all projects together.

**GitHub Packages — dependency resolution:** Add to `pom.xml` `<repositories>`:
```xml
<repository>
  <id>github</id>
  <url>https://maven.pkg.github.com/casehubio/*</url>
  <snapshots><enabled>true</enabled></snapshots>
</repository>
```
CI must use `server-id: github` + `GITHUB_TOKEN` in `actions/setup-java`.

**Cross-project SNAPSHOT versions:** `casehub-ledger` and `casehub-work` modules are `0.2-SNAPSHOT` resolved from GitHub Packages. Declare in `pom.xml` properties and `<dependencyManagement>` — no hardcoded versions in submodule poms.

