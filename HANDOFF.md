# HANDOFF — casehub-claudony

## Last Session

Completed three issues from the #205 fleet manager queue:

**#215 — conversation-id capture.** Instead of extracting conversation IDs post-hoc from running Claude sessions, assigned them at creation time via `claude --session-id <uuid>`. Resume uses `claude -r <uuid>`. `TmuxSessionOperations.create()` generates a UUID, appends `--session-id`, stores it immediately. `conversationId()` returns the assigned UUID for any created session. Full round-trip test: create → suspend → resume preserves conversation ID. 16 TmuxSessionOperations tests + 14 AgentSessionManager tests green.

**#216 — shared file coordination (D11 resolved).** Added `WorkingDirPolicy` enum (EXCLUSIVE, SHARED_READ, BRANCH_ISOLATED) with conflict detection in `AgentSessionManager.acquireSession()`. Default is EXCLUSIVE — a second active session on the same workingDir throws `WorkingDirConflictException`. Worker provisioning (`openWorkerSession`) uses SHARED_READ since CaseHub-coordinated workers legitimately share working directories. D11 decision updated in `specs/issue-205-llm-fleet-manager/decisions.md`. 262 casehub tests green.

**#217 — CDI deployment fix.** Replaced 70+ individual engine bean exclusions with glob patterns (`io.casehub.engine.internal.**`, `io.casehub.engine.scheduler.**`, `io.casehub.engine.trust.**`, `io.casehub.connectors.**`, `io.casehub.neocortex.**`). New engine SNAPSHOT beans get excluded automatically. Added `NoOpProvisionerConfigRegistry @DefaultBean` in test sources to satisfy `CompositeProviderConfigSource` injection. Also excluded new qhorus/ledger beans: `qhorus.runtime.store.jpa.**`, A2A/AgentCard/CausalGraph resources, `QhorusPushWebSocket`, `QhorusInboundCurrentPrincipal`, `DefaultOutcomeRecorder`, `AuditedInterceptor`, `TestWorkerProvisioner`. SmokeTest + 72 app tests green. Remaining classloading failures in engine-dependent test classes (`ClaudonyLedgerEventCaptureSignalTest`, `AgentCaseCompletionTest`, `ClaudonyCaseChannelProviderPostgresIT`) need engine SNAPSHOT rebuilt from source — engine and platform repos are in this slot (slot 202).

## Immediate Next Step

Rebuild engine SNAPSHOT from source in this slot, then continue with #218 (engine convergence) and #219 (ConfigMapping agent pool). The engine repo is at `/Users/mdproctor/claude/casehub/slots/202/engine/`, platform at `/Users/mdproctor/claude/casehub/slots/202/platform/`.

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn install -DskipTests -q -f /Users/mdproctor/claude/casehub/slots/202/engine/pom.xml
```

After engine rebuild, verify classloading failures are resolved:
```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl app -Denforcer.skip=true -f /Users/mdproctor/claude/casehub/slots/202/claudony/pom.xml
```

Then update `CasehubEnabledProfile` and `CompletionTestProfile` exclude-types overrides to match the new glob-based default profile (per `engine-cdi-exclude-types-sync` protocol).

## Queue

Position 5/8. #205, #213, #214, #215, #216 done. Active on #217 (partially done — CDI fixed, classloading remains). Then #218 (engine convergence), #219 (ConfigMapping).

## Frontend Build

Frontend npm build fails with `portal:` protocol error on npm 11.x. Not blocking Java tests (enforcer skipped with `-Denforcer.skip=true`). Investigate npm compatibility or use slot 194's node_modules.

## References

- `specs/issue-205-llm-fleet-manager/2026-09-21-llm-fleet-manager-design.md`
- `specs/issue-205-llm-fleet-manager/decisions.md` (D1–D11, all captured)
- `plans/2026-09-21-llm-fleet-manager.md`
