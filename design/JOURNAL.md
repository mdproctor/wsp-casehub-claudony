# Design Journal — issue-138-qhorus-184-cleanup

Three issues in sequence on this branch:

- **#138 (XS)** — Remove `MessageResult` stub + fix `sendMessage()` call sites (Qhorus #184 migration complete)
- **#124 (S)** — Feed cursor `?after=<id>` for `GET /api/mesh/feed` (needs Qhorus `getFeed()` cursor too)
- **#135 (S)** — Add `correlationId` to `postToChannel()` SPI in casehub-engine; remove content-coupling parse in Claudony
