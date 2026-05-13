# Handover — 2026-05-13

**Head commit:** `8b1cde8` — @TestSecurity removal noted in CLAUDE.md  
**Branch:** `main`, all tests passing (479, zero failures)

---

## What happened this session

**Issues closed:** #109 (package rename — verified + stale comment cleanup), #91 (already done — closed), #108 (SSE strategy integration tests), #110 (register non-trigger test + @TestSecurity removal from HybridDefaultConfigTest), #111 (@TestSecurity removal from RegistryHooksStrategyIntegrationTest)

**#108 SSE integration tests landed:**
- `RegistryHooksStrategyIntegrationTest` (4 tests) — strategy type, updateStatus push, remove push, register does NOT push
- `HybridDefaultConfigTest` (1 test) — strategy type is hybrid when config=hybrid; no @TestSecurity
- Both use `@QuarkusTestProfile` inner classes to override `claudony.case-worker-update`
- AtomicInteger snapshotFn distinguishes initial snapshot from registry-triggered push by content, not just count

**Key discovery:** `@TestSecurity` is silently ignored on `@QuarkusTest` classes that never touch an HTTP endpoint. Cargo-culted into two new CDI-only test classes from the HTTP-exercising tests around them. Caught by code review. Removed from both; added platform protocol PP-20260513-7c227e and garden entry GE-20260513-3c1a03.

**Garden:** GE-20260513-3c1a03 (@TestSecurity silent no-op), GE-20260513-4c4205 (AtomicInteger snapshotFn technique)  
**Protocol:** PP-20260513-7c227e (quarkus-test-security-http-only) → parent repo

---

## Test count

**479 passing, 0 failures.** 4 in `claudony-core` + 134 in `claudony-casehub` + 341 in `claudony-app`.

---

## Open issues

- `#111` — closed this session
- `#108` — closed this session
- Remaining open: #86 (agent mesh epic), #94 (causal chain, blocked on engine), #99 epic (#98, #100, #101, #102 — blocked on Qhorus #131), #105 (MCP endpoint separation)

*Full list: `gh issue list --repo casehubio/claudony --state open`*

---

## Immediate next

Pick up #86 (agent mesh infrastructure epic — self-contained, no upstream blockers) or wait for engine/Qhorus unblocking on #94/#98.

---

## Key files

- Blog: `wksp/blog/2026-05-13-mdp01-silent-annotation-and-sse-causality.md`
- Protocol: `~/claude/casehub/parent/docs/protocols/quarkus-test-security-http-only.md`
- Garden: `~/.hortora/garden/jvm/GE-20260513-3c1a03.md`, `GE-20260513-4c4205.md`
