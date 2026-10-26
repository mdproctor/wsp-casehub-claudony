# @McpDomain Migration Cleanup — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #204 — feat: unified API generation — migrate to @McpDomain SPI with JAX-RS annotations
**Issue group:** #204

**Goal:** Delete deprecated hand-written REST resources that are now superseded by the generator-produced REST resources from @McpDomain SPIs, and resolve the FleetResource path conflict.

**Architecture:** Platform#295 landed the `GraphQLResolverProcessor` APT, which generates JAX-RS REST resources from `@McpDomain`/`@PlatformQuery`/`@PlatformMutation` SPIs. Claudony already has the generator active (`-AgenerateGraphQL=false` — REST only) with a `domainFilter` covering all 10 domains. Generated resources exist at `/api/claudony/*` paths. Three hand-written resources are now redundant and marked deprecated or annotated as hand-written for reasons that no longer hold.

**Tech Stack:** Java 21, Quarkus 3.32.2, Maven, casehub-platform graphql-generator APT

## Global Constraints

- Package is `io.casehub.claudony`, not `dev.claudony`
- Use `ide_refactor_safe_delete` for removing Java classes, `ide_search_text` for verifying no remaining references
- Run `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl app` after each batch to verify
- Generated resources live in `app/target/generated-sources/annotations/io/casehub/claudony/server/rest/`
- Generated resources do NOT carry `@Authenticated` or `@RolesAllowed` — this is a platform generator gap, not in scope

---

## Batch 1: Delete deprecated PoolResource + supporting types

### Task 1: Delete PoolResource and its test

**Files:**
- Delete: `app/src/main/java/io/casehub/claudony/server/fleet/PoolResource.java`
- Delete: `app/src/test/java/io/casehub/claudony/server/fleet/PoolResourceTest.java`
- Delete: `app/src/main/java/io/casehub/claudony/server/fleet/CapacityUpdate.java`
- Delete: `app/src/main/java/io/casehub/claudony/server/fleet/ScalingConfigUpdate.java`

**Interfaces:**
- Consumes: `PoolService` (injected — still used by `ClaudonyPoolApi`)
- Produces: nothing — this is a deletion

- [ ] **Step 1: Verify no references to CapacityUpdate or ScalingConfigUpdate outside PoolResource**

Run `ide_search_text` for each type. Expected: only `PoolResource.java` and the record file itself.

- [ ] **Step 2: Verify no frontend references to `/api/pools`**

Run `ide_search_text` for `/api/pools` in `*.ts,*.html` files. Expected: no matches.

- [ ] **Step 3: Delete PoolResource.java**

Use `ide_refactor_safe_delete` on `PoolResource`.

- [ ] **Step 4: Delete CapacityUpdate.java and ScalingConfigUpdate.java**

Use `ide_refactor_safe_delete` on each record.

- [ ] **Step 5: Delete PoolResourceTest.java**

Use `ide_refactor_safe_delete` on `PoolResourceTest`.

- [ ] **Step 6: Delete CredentialRoleAugmentorTest.java**

This test exclusively tests auth on `/api/pools` paths that no longer exist.
Use `ide_refactor_safe_delete`.

- [ ] **Step 7: Run tests**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl app -Dtest=ClaudonyPoolApiTest
```

Expected: PASS — generated endpoints at `/api/claudony/pools` still work.

- [ ] **Step 8: Commit**

```bash
git add -A
git commit -m "feat(#204): delete deprecated PoolResource and supporting types

Remove PoolResource (/api/pools), CapacityUpdate, ScalingConfigUpdate,
PoolResourceTest, and CredentialRoleAugmentorTest. All pool endpoints are
now served by the generated ClaudonyPoolsResource at /api/claudony/pools.

Refs #204

Co-Authored-By: Claude Opus 4.6 (1M context) <noreply@anthropic.com>"
```

### Task 2: Delete AgentPoolResource and its test

**Files:**
- Delete: `app/src/main/java/io/casehub/claudony/server/fleet/AgentPoolResource.java`
- Delete: `app/src/test/java/io/casehub/claudony/server/fleet/AgentPoolResourceTest.java`
- Modify: `casehub/src/main/java/io/casehub/claudony/casehub/fleet/AgentPoolStatus.java` — update javadoc comment

**Interfaces:**
- Consumes: `ClaudonyAgentBackend.poolStatus()` — still used elsewhere
- Produces: nothing — this is a deletion

- [ ] **Step 1: Verify no references to AgentPoolResource outside its test**

Run `ide_find_references` on `AgentPoolResource`. Expected: only test file and itself.

- [ ] **Step 2: Delete AgentPoolResource.java**

Use `ide_refactor_safe_delete`.

- [ ] **Step 3: Delete AgentPoolResourceTest.java**

Use `ide_refactor_safe_delete`.

- [ ] **Step 4: Update AgentPoolStatus javadoc**

Remove the `{@code GET /api/agent-pools}` reference from the javadoc comment in `AgentPoolStatus.java`. The endpoint no longer exists.

- [ ] **Step 5: Run tests**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl app -Dtest=SmokeTest
```

Expected: PASS — no compilation errors, app starts.

- [ ] **Step 6: Commit**

```bash
git add -A
git commit -m "feat(#204): delete deprecated AgentPoolResource

Remove AgentPoolResource (/api/agent-pools) and AgentPoolResourceTest.
Update AgentPoolStatus javadoc to remove stale endpoint reference.

Refs #204

Co-Authored-By: Claude Opus 4.6 (1M context) <noreply@anthropic.com>"
```

## Batch 2: Resolve FleetResource path conflict

### Task 3: Remove FleetResource — let generated resource handle fleet execute

The hand-written `FleetResource` at `/api/claudony/fleet/execute` accepts `text/plain` YAML body. The generated `ClaudonyFleetResource` maps the same path but takes YAML as a `@QueryParam`. Both exist simultaneously.

The hand-written version is correct (YAML should be a body, not a query param). The fix is to remove `FleetResource` and update the `ClaudonyFleetApi` SPI method signature so the generator produces the correct binding. If the generator cannot express `text/plain` body consumption, keep `FleetResource` with `@HandWrittenEndpoint` and remove `claudony/fleet` from the `domainFilter` to prevent the generated duplicate.

**Files:**
- Modify: `app/src/main/java/io/casehub/claudony/server/api/ClaudonyFleetApi.java` — check if body param can be expressed
- Possibly delete: `app/src/main/java/io/casehub/claudony/server/fleet/FleetResource.java`
- Possibly modify: `app/pom.xml` — remove `claudony/fleet` from `domainFilter`

**Interfaces:**
- Consumes: `FleetScriptService.execute(String yaml)`
- Produces: `FleetScriptResult`

- [ ] **Step 1: Read ClaudonyFleetApi to understand current SPI signature**

```java
// Check if the yaml parameter has @QueryParam or is a bare String (body)
```

- [ ] **Step 2: Determine if generator supports body parameters**

Check `GraphQLResolverProcessor` — does it map unannotated String params as body or query? If body, we can fix the SPI. If query (convention), we need `@HandWrittenEndpoint` to stay.

- [ ] **Step 3a: If generator supports body → fix SPI and delete FleetResource**

Remove any `@QueryParam` from the `yaml` parameter in `ClaudonyFleetApi.executeFleetScript()`. Rebuild to verify the generated resource uses body. Delete `FleetResource`.

- [ ] **Step 3b: If generator doesn't support body → keep FleetResource, remove from domainFilter**

Remove `claudony/fleet` from the `-AdomainFilter` compiler arg in `app/pom.xml`. This prevents the generator from producing a conflicting `ClaudonyFleetResource`. Keep `FleetResource` as the sole handler.

- [ ] **Step 4: Rebuild and run tests**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl app -Dtest=ClaudonyFleetApiTest
```

Expected: PASS.

- [ ] **Step 5: Commit**

```bash
git add -A
git commit -m "feat(#204): resolve FleetResource path conflict

[describe which approach was taken]

Refs #204

Co-Authored-By: Claude Opus 4.6 (1M context) <noreply@anthropic.com>"
```

## Batch 3: Documentation and follow-ups

### Task 4: Update CLAUDE.md and file follow-up issues

**Files:**
- Modify: `CLAUDE.md` — update test count, remove deprecated resource references
- Modify: `docs/CHANNELS.md` or other docs referencing `/api/pools` or `/api/agent-pools` if any

- [ ] **Step 1: Search for stale path references in docs**

Run `ide_search_text` for `/api/pools` and `/api/agent-pools` in `*.md` files. Update or remove references.

- [ ] **Step 2: Update test count in CLAUDE.md**

Subtract deleted tests from the baseline count. `PoolResourceTest` had 9 tests, `AgentPoolResourceTest` had 2, `CredentialRoleAugmentorTest` had 4 = 15 fewer tests.

- [ ] **Step 3: File follow-up issue for auth on generated resources**

The deprecated `PoolResource` had `@Authenticated` + `@RolesAllowed("admin")` on mutations. The generated `ClaudonyPoolsResource` has neither. File issue on `casehubio/platform` for the generator to support `@Authenticated` and `@RolesAllowed` propagation from SPI annotations or configuration.

- [ ] **Step 4: Run full test suite**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test
```

Expected: all pass.

- [ ] **Step 5: Commit**

```bash
git add -A
git commit -m "docs(#204): update CLAUDE.md test counts and stale references

Refs #204

Co-Authored-By: Claude Opus 4.6 (1M context) <noreply@anthropic.com>"
```

## References

- [ClaudonyPoolApi.java] — existing @McpDomain SPI for pools
- [ClaudonyFleetApi.java] — existing @McpDomain SPI for fleet
- [PoolResource.java] — deprecated, to delete
- [AgentPoolResource.java] — deprecated, to delete
- [FleetResource.java] — hand-written, path conflict with generated
- [GraphQLResolverProcessor] — platform generator (casehubio/platform)
- [GitHub casehubio/platform#295] — unified API generation epic (closed/completed)
- [GitHub casehubio/claudony#204] — this issue
