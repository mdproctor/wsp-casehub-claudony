# Design: Align Agent Pool Canonical Layer with Platform YAML Language

**Epic:** #235
**Children:** #236 (YAML parser), #237 (annotation scanner)
**Date:** 2026-09-28

## Context

The agent pool canonical layer (#231–#234) provides three frontends over a single model (`AgentPoolDefinition`): a fluent DSL, YAML parsing, and annotation scanning. These were built before the platform YAML language gained typed parameters (`StepDefinition`/`StepParameter`/`StepParameterType`), validation (`StepValidator`), JSON Schema generation (`SchemaEmitter`), and classpath-scanned plugins (`@StepPlugin` + APT processor + `AptPluginSource`).

The pool's YAML parser uses raw Jackson `ObjectMapper(YAMLFactory)` with manual `Map<String, Object>` traversal and custom helper methods (`stringValue()`, `intValue()`, `parsePolicy()`, `parseEviction()`). The annotation scanner uses runtime reflection (`Class.getAnnotation()`) with a custom scanning loop. Both work, but diverge from platform patterns — missing typed validation, JSON Schema generation (IDE autocomplete), and build-time discovery.

## Goal

Align the two divergent frontends with the platform's YAML language infrastructure:
1. YAML parser → typed `StepDefinition` + `StepValidator` for validation
2. Annotation scanner → `@PoolDefinition` record + APT processor for schema generation and classpath discovery

The fluent DSL and `AgentPoolConfigBridge` are unchanged — they're pure Java and don't benefit from YAML infrastructure alignment.

## Non-Goals

- Executable step integration — pool definitions are declarative config, not executable steps. No `@Execute`, `StepResult`, or `StepAction`.
- DesiredState/ops integration — depends on casehubio/casehub-desiredstate#152, separate work.
- `AgentPoolConfigBridge` migration — marginal improvement, not worth the churn.

---

## Part 1: YAML Parser Migration (#236)

### Current State

`AgentPoolYamlParser` parses this YAML structure:

```yaml
agent-pools:
  code-reviewer:
    working-dir: /workspace/reviews
    policy: SHARED_READ
    command: claude --model opus
    pool:
      min-active: 2
      max-active: 10
      eviction: memory-weighted
```

Using: `ObjectMapper(YAMLFactory).readValue(yaml, Map.class)` → manual map traversal with `stringValue()`, `intValue()`, `parsePolicy()`, `parseEviction()`.

### Target State

Define a static `StepDefinition` describing the pool parameter schema:

| Parameter | Type | Required | Default | Allowed Values |
|-----------|------|----------|---------|----------------|
| `working-dir` | STRING | no | — | — |
| `policy` | STRING | no | `EXCLUSIVE` | EXCLUSIVE, SHARED_READ, BRANCH_ISOLATED |
| `command` | STRING | no | — | — |
| `pool.min-active` | INTEGER | no | `0` | — |
| `pool.max-active` | INTEGER | no | `10` | — |
| `pool.eviction` | STRING | no | `MEMORY_WEIGHTED` | MEMORY_WEIGHTED, LRU |

The `name` is the YAML map key, not a parameter — it's always required and doesn't go through StepValidator.

### Processing Pipeline

1. **Parse YAML** — Jackson `ObjectMapper(YAMLFactory)` produces `Map<String, Object>` (unchanged; platform yaml-core doesn't own YAML parsing)
2. **Extract `agent-pools` section** — get the named map entries (unchanged)
3. **Flatten** — for each pool entry, merge the `pool:` sub-map into the parent map with `pool.` prefix (e.g., `pool.min-active`). This transforms the nested structure into StepValidator's flat model.
4. **Validate** — call `StepValidator.validateStep(name, flatMap, poolStepDefinition)`. Returns typed error messages on violations (wrong type, unknown enum value, missing required field).
5. **Extract** — read validated values using `StepParameterType` coercion. Enum fields use `allowedValues` constraint on the StepParameter, replacing manual `parsePolicy()`/`parseEviction()`.
6. **Build** — construct `AgentPoolDefinition` via the fluent builder (unchanged)

### What Changes

| Before | After |
|--------|-------|
| `stringValue(map, "policy")` | `StepParameterType.STRING.validate(value)` + `allowedValues` check |
| `intValue(map, "min-active")` | `StepParameterType.INTEGER.validate(value)` |
| `parsePolicy(string)` | `WorkingDirPolicy.valueOf(validated_string)` — `allowedValues` ensures it's valid |
| `parseEviction(string)` | `EvictionStrategy.valueOf(validated_string)` — `allowedValues` ensures it's valid |
| No validation feedback | `StepValidator.validateStep()` returns structured error list |

### What Gets Deleted

- `stringValue(Map, String)` helper
- `intValue(Map, String)` helper
- `parsePolicy(String)` helper
- `parseEviction(String)` helper

### New Dependency

`casehub-platform-yaml-core` (`io.casehub:casehub-platform-yaml-core:${casehub.version}`) added to `claudony-casehub/pom.xml`. This is a zero-dependency library (no transitive pull).

### YAML Surface

**Unchanged.** The YAML format stays exactly the same — only the internal parsing and validation pipeline changes.

### Test Strategy

- Existing `AgentPoolYamlParserTest` tests pass unchanged (same behavior, different internals)
- Add validation error tests: wrong type for `min-active`, invalid policy value, invalid eviction value
- Add test verifying the `StepDefinition` schema matches the expected parameter structure

---

## Part 2: Annotation Scanner Migration (#237)

### Current State

Two runtime annotations on classes:

```java
@PooledAgent(name = "code-reviewer", workingDir = "/workspace/reviews")
@AgentPool(minActive = 2, maxActive = 10)
public class CodeReviewerAgent {}
```

Scanned by `AgentPoolAnnotationScanner` using `Class.getAnnotation()` at runtime.

### Target State

Single `@PoolDefinition` annotation on a **record**:

```java
@PoolDefinition("code-reviewer")
public record CodeReviewerPool(
    @Optional String workingDir,
    @Optional WorkingDirPolicy policy,
    @Optional String command,
    @Optional int minActive,
    @Optional int maxActive,
    @Optional EvictionStrategy eviction
) {}
```

### APT Processor: PoolDefinitionProcessor

Lives in `claudony-casehub` (not the platform). Follows the same structural pattern as `StepPluginProcessor`:

1. Scans for `@PoolDefinition`-annotated records
2. Validates: must be a record, must have valid field types
3. Emits:
   - **JSON Schema** → `META-INF/pool-definitions/<name>.schema.json`
   - **Registry manifest** → `META-INF/pool-definitions/<name>.json`
4. Does NOT emit a binder class (no execution)

**Manifest format:**

```json
{
  "name": "code-reviewer",
  "recordClass": "com.example.CodeReviewerPool",
  "schemaResource": "META-INF/pool-definitions/code-reviewer.schema.json"
}
```

**Schema format** (same structure as `SchemaEmitter`):

```json
{
  "$schema": "https://json-schema.org/draft/2020-12/schema",
  "type": "object",
  "properties": {
    "working-dir": { "type": "string" },
    "policy": { "type": "string", "enum": ["EXCLUSIVE", "SHARED_READ", "BRANCH_ISOLATED"] },
    "command": { "type": "string" },
    "min-active": { "type": "integer" },
    "max-active": { "type": "integer" },
    "eviction": { "type": "string", "enum": ["MEMORY_WEIGHTED", "LRU"] }
  },
  "required": [],
  "additionalProperties": false
}
```

Note: enum values for `policy` and `eviction` are derived from the field type's enum constants at compile time. The schema uses a flat structure (no nested `pool:` — the schema describes individual pool entries, not the full `agent-pools:` document).

### Runtime Discovery: PoolDefinitionSource

Analogous to `AptPluginSource`. Scans `META-INF/pool-definitions/*.json` manifests on the classpath, loads the schema, constructs `AgentPoolDefinition` via the builder, and registers in `AgentPoolDefinitionRegistry`.

### Annotation API

**New:**
- `@PoolDefinition(String value)` — annotation with pool name (like `@StepPlugin(String value)`)
- Uses `@Required` / `@Optional` from `io.casehub.yaml.plugin.api` (reused, not duplicated)

**Deleted:**
- `@PooledAgent` — replaced by `@PoolDefinition`
- `@AgentPool` — merged into the record fields
- `AgentPoolAnnotationScanner` — replaced by APT + `PoolDefinitionSource`

### New Dependencies

- `casehub-platform-yaml-plugin-api` — for `@Required`/`@Optional` annotations (reused)
- `casehub-platform-yaml-plugin-processor` — APT processor reference (for pattern, not dependency — Claudony has its own processor)

### Test Strategy

- Existing `AgentPoolAnnotationScannerTest` tests rewritten to use `@PoolDefinition` records
- APT processor tests (compile-time): verify schema and manifest generation
- `PoolDefinitionSource` tests: verify classpath scanning and registry population
- Equivalence test: `@PoolDefinition` record + builder DSL + YAML all produce the same `AgentPoolDefinition`

---

## Module Impact

| Module | Changes |
|--------|---------|
| `claudony-casehub` | New dep: `casehub-platform-yaml-core`. YAML parser refactored. New `@PoolDefinition` annotation, `PoolDefinitionProcessor`, `PoolDefinitionSource`. Old annotations + scanner deleted. |
| `claudony-app` | None — consumers use `AgentPoolDefinitionRegistry` (unchanged API) |
| `claudony-core` | None |
| Platform | None — no changes to platform repos |

## Unchanged Components

- `AgentPoolDefinition` — immutable model with `AgentConfig` + `PoolConfig` records
- `AgentPoolDefinition.Builder` — fluent DSL
- `AgentPoolDefinitionRegistry` — `ConcurrentHashMap`-backed registry
- `AgentPoolConfigBridge` — ops ProviderConfig bridge
- `WorkingDirPolicy`, `EvictionStrategy` — enum types
- `AgentPoolHealth`, `AgentPoolStatus`, `AgentPoolConfig` — fleet management types
- `AgentSessionManagerConfig` — session manager config record

## Benefits

1. **Typed validation** — wrong types, invalid enum values, and constraint violations caught with structured error messages instead of silent defaults or raw exceptions
2. **JSON Schema generation** — IDE autocomplete and validation for pool YAML sections and pool definition records
3. **Build-time discovery** — pool definitions discovered via classpath scanning of APT-generated manifests, not runtime reflection
4. **Consistent error messages** — `StepValidator` produces the same error format used across the platform
5. **Reduced manual code** — all `stringValue`/`intValue`/`parsePolicy`/`parseEviction` helpers eliminated
6. **Platform coherence** — pool config parsing follows the same patterns as every other YAML-configured component in the platform

## References

- `io/casehub/yaml/core/step/StepDefinition.java` — platform typed parameter model
- `io/casehub/yaml/core/step/StepParameter.java` — typed parameter with validation metadata
- `io/casehub/yaml/core/step/StepParameterType.java` — type system (STRING, INTEGER, etc.)
- `io/casehub/yaml/core/step/StepValidator.java` — validates params against definitions
- `io/casehub/yaml/plugin/api/StepPlugin.java` — platform plugin annotation
- `io/casehub/yaml/plugin/processor/StepPluginProcessor.java` — APT processor pattern
- `io/casehub/yaml/plugin/processor/SchemaEmitter.java` — JSON Schema generation
- `io/casehub/yaml/step/catalog/AptPluginSource.java` — classpath scanning pattern
- `io/casehub/claudony/casehub/fleet/AgentPoolYamlParser.java` — current parser (being replaced)
- `io/casehub/claudony/casehub/fleet/AgentPoolAnnotationScanner.java` — current scanner (being replaced)
- `io/casehub/claudony/casehub/fleet/AgentPoolDefinition.java` — model (unchanged)
- `io/casehub/claudony/casehub/fleet/AgentPoolConfigBridge.java` — dotted key convention reference
- Issue #235 — epic
- Issues #236, #237 — child issues
