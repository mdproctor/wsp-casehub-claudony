# Handover — 2026-05-23

**Head commit (project):** `8000e35` — refactor(casehub): use first-class correlationId and deadline SPI params — Closes #135
**Branch:** `main` — both repos. Clean close (branch `issue-138-qhorus-184-cleanup` marked closed, deletion due 2026-06-06).

---

## Last Session

Closed #138, #124, and #135 on a single branch. #138: removed MessageResult stub
and fixed sendMessage() call sites after Qhorus #184 migration. #124: redesigned
getFeed() from per-channel bucketing to global DESC scan — found and fixed
JpaMessageStore SQL LIMIT OOM bug (Panache .list() vs .find().page()), added V11
composite index on message(channel_id, id). #135: added correlationId and deadline
as first-class params to postToChannel() SPI (cross-repo: engine#343 + qhorus#192),
eliminating content-coupling JSON parse. Squashed 12 → 5 commits. 508 tests passing.

## Immediate Next Step

Both repos on `main`. CI "Build and Publish" is failing on `casehub-testing:0.2-SNAPSHOT`
not found in GitHub Packages — engine CI is deploying now. Once engine CI is green,
re-trigger Claudony CI: `gh workflow run maven.yml --repo casehubio/claudony`. Verify green.

---

## What's Left

- **#125** — SSE `Last-Event-ID` reconnect for `/api/mesh/events` · M · Med
- **#131** — ChannelEventBus-driven true push (replace 500ms tick) · M · Med
- **qhorus#175–177** — DTO/mapper/transaction cleanup · M · Low
- **qhorus#181** — ChannelGateway not re-initialized on restart · M · Med

---

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| #102 | Fleet-aware channel backend registration — push to all active nodes | L | High | Core epic follow-on; new branch |
| #117 | `HumanParticipatingChannelBackend` — bidirectional channel posting via backend | M | Med | Fully unblocked |
| #131 | ChannelEventBus-driven true push (replace 500ms tick in channelEvents) | M | Med | Investigation needed: cross-thread emit fix |

---

## Key references

- Specs: `docs/specs/2026-05-22-feed-desc-global-scan-design.md`, `docs/specs/2026-05-23-postToChannel-correlationId-deadline-design.md`
- Garden: GE-20260523-06e8b6 (Panache .list() OOM — SQL LIMIT not applied)
- BUGS-AND-ODDITIES: #21 (casehub-engine jar self-indexes via embedded application.properties)
- Test baseline: 4 core + 133 casehub + 371 app = 508 (2026-05-23)
- Blog: `2026-05-23-mdp03-feed-that-couldnt-show-new-messages.md`
