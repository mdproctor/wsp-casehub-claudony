# HANDOFF — casehub-claudony

## Last Session

Completed #219 (ConfigMapping agent pool). Ran a full 4-dimension audit of the entire #205 branch (40 files, ~1900 lines). Created 7 fix issues from audit findings, grouped under epic #227.

**#219 — ConfigMapping agent pool (completed).** Replaced hardcoded `AgentSessionManagerConfig(0, 10)` in `ClaudonyAgentBackend` with injectable `AgentPoolConfig` (`@ConfigMapping(prefix = "claudony.agent-pool")`). Properties: `min-active` (default 0), `max-active` (default 10). Test updated, new test `poolStatus_reflectsConfiguredMinMax` added. CLAUDE.md documented.

**Branch audit — 4 parallel audits completed:**
- CDI wiring: all PASS — beans discoverable, no exclusion conflicts, no ambiguity
- End-to-end flow: 4/6 paths COMPLETE, 2 PARTIAL (close() leak, BRANCH_ISOLATED unimplemented)
- Code quality: 1 CRITICAL, 2 HIGH, 3 MEDIUM findings
- Test coverage: 80 tests across 6 files, but 3 significant gaps identified

## Immediate Next Step

Start fixing audit issues. #220 is active — the critical pool leak in `TmuxAgentSession.close()`. All 7 issues are XS/Low. Run `work continue`.

## Audit Issues (epic #227)

### Runtime correctness (must fix)
| # | Title | Key detail |
|---|-------|-----------|
| #220 | `close()` doesn't release session to pool | Session stays ACTIVE, pool leaks to exhaustion. Fix: call `suspendSession()` in `close()` |
| #221 | `terminate()` bypasses `close()` | `ClaudonyWorkerProvisioner.terminate()` calls `destroySession()` directly, skipping close lifecycle |
| #222 | Eviction score additive vs multiplicative | `idle + mem/10` but design D9 says `idle x memWeight`. Memory barely matters for short idle |

### Defensive guards (should fix)
| # | Title | Key detail |
|---|-------|-----------|
| #223 | `BRANCH_ISOLATED` declared but unimplemented | Enum value exists, acquireSession treats it as SHARED_READ silently. Add UnsupportedOperationException guard |
| #224 | `AgentPoolResource` no auth | Other fleet endpoints have auth annotations. Consistency gap |

### Test coverage (should fix)
| # | Title | Key detail |
|---|-------|-----------|
| #225 | `openWorkerSession()` untested | Primary production method (called by provisioner) has zero direct unit tests |
| #226 | `AgentPoolResource` no QuarkusTest | REST endpoint `GET /api/agent-pools` has zero test coverage |

### Sequencing
Fix #220 first (critical leak). #221 depends on #220's close() behavior. Rest in any order.

## Pre-Existing Failures

- Maven in slot 202 gets 401 from GitHub Packages — expired token. Clear `_remote.repositories` and `.lastUpdated` files from slot `.m2`, or re-authenticate. IntelliJ diagnostics work fine for compilation verification.
- Engine: pre-existing checkstyle errors and neocortex build failures (not caused by this branch)

## Queue

Position 9/15. #205–#219 done. #220–#226 pending (audit fixes). Next: #220 (close-pool-leak).

## Key Discoveries

- **Pool leak is the showstopper:** Without #220, every worker session that completes leaks from the pool. In a production scenario with repeated case execution, this would exhaust `maxActive` (default 10) and halt all provisioning.
- **Design spec D9 vs implementation mismatch:** Eviction formula doesn't match the documented design. The additive formula makes memory weight negligible for sessions with < 100s idle time.
- **BRANCH_ISOLATED is a future feature:** The enum value was added for completeness per design spec D11, but no implementation exists. A guard is safer than silent no-op.

## References

- `specs/issue-205-llm-fleet-manager/2026-09-21-llm-fleet-manager-design.md`
- `specs/issue-205-llm-fleet-manager/decisions.md` (D1–D11)
- `plans/2026-09-21-llm-fleet-manager.md`
- Epic #227 — audit fixes
