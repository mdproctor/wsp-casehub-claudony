# Handover — 2026-05-22

**Head commit (project):** `e3ec65f` — docs(claude): add Name field for write-blog project tagging
**Branch:** `main` — both repos. Clean close.

---

## Last Session

Closed #136 and #137: test cleanup on `MeshResourceTest` (comment accuracy, catch
block parity, ObjectMapper injection, stronger multi-channel assertion). Discovered
Qhorus #184 mid-migration had broken Claudony test compilation via a stale runtime
jar; fixed with a test-scope `MessageResult` stub tracked by #138. Added
`**Name:** claudony` to workspace CLAUDE.md and fixed a capitalisation bug in one
blog entry; all 20 workspace blog entries are now published to mdproctor.github.io.
507 tests passing.

## Immediate Next Step

Both repos on `main`. Pick next issue: #102 (fleet-aware backend registration)
or #117 (`HumanParticipatingChannelBackend`) — both recommended by previous handover.

---

## What's Left

- **#124** — feed cursor `?after=<id>` support — deferred, needs Qhorus `getFeed()` change · S · Low
- **#125** — SSE `Last-Event-ID` reconnect for `/api/mesh/events` · M · Med
- **#131** — ChannelEventBus-driven true push (replace 500ms tick) · M · Med
- **#135** — Add `correlationId` param to `postToChannel()` SPI in casehub-engine · S · Med · cross-repo
- **#138** — Remove `MessageResult` stub once Qhorus #184 migration completes · XS · Low
- **qhorus#175–177** — DTO/mapper/transaction cleanup · M · Low
- **qhorus#181** — ChannelGateway not re-initialized on restart · M · Med

---

## What's Next

*Unchanged — `git show HEAD~1:HANDOFF.md`*

---

## Key references

- ADRs: `docs/adr/0006-channel-backend-registration-timing.md`, `0007-sse-channel-delivery-mechanism.md`
- Garden: GE-20260522-e6eb70 (Maven phantom class compile error from deleted peer-repo class)
- Test baseline: 4 core + 134 casehub + 369 app = 507 (2026-05-22)
- Blog: `blog/2026-05-22-mdp07-three-fixes-phantom-class.md`
