---
layout: post
title: "Three Things Platform Alignment Actually Buys You"
date: 2026-09-28
entry_type: note
subtype: diary
projects: [casehubio/claudony]
tags: [platform, yaml, typed-parameters, annotation-processing, design]
series: issue-235-align-pool-yaml-platform
---

# Three Things Platform Alignment Actually Buys You

The agent pool canonical layer had three frontends — fluent DSL, YAML parsing, annotations — all producing the same `AgentPoolDefinition`. They worked. The YAML parser extracted values from Jackson maps with manual helpers. The annotation scanner reflected on `@PooledAgent` at runtime. Both were correct and tested.

Then the platform YAML language gained typed parameters, validation, JSON Schema generation, and build-time annotation processing. The pool layer predated all of it. The question was whether alignment was worth the churn.

It was. Here's what it actually bought.

## Typed validation replaces manual extraction

The old parser had four helpers doing the same job the platform already does:

```java
// Before — manual extraction, per field
private static String stringValue(Map<String, Object> map, String key) {
    var value = map.get(key);
    return value != null ? value.toString() : null;
}

private static WorkingDirPolicy parsePolicy(String value) {
    return WorkingDirPolicy.valueOf(value.toUpperCase().replace('-', '_'));
}
```

A wrong enum value threw a raw `IllegalArgumentException` from `valueOf` — no context about which field, which pool definition, or what was expected. Pass `min-active: "not-a-number"` and it silently treated it as null.

After alignment, the pool schema is a `StepDefinition` with typed `StepParameter` entries:

```java
inputs.put("policy", new StepParameter(
        StepParameterType.STRING, false, "EXCLUSIVE",
        Arrays.stream(WorkingDirPolicy.values()).map(Enum::name).toList(),
        null, "Working directory concurrency policy"));
```

`StepValidator.validateStep()` checks types, enforces `allowedValues`, and returns structured errors: `"worker: input 'policy' value 'INVALID' not in allowedValues [EXCLUSIVE, SHARED_READ, BRANCH_ISOLATED]"`. Every field validated against its declared type before extraction even begins. The four manual helpers are gone.

The parser itself is now a thin wrapper: flatten the YAML map, validate, extract typed values. The domain knowledge — what fields exist, what types they are, what values are legal — lives in the schema declaration, not scattered across extraction logic.

## JSON Schema falls out of the annotation model

The old `@PooledAgent` and `@AgentPool` annotations were scanned at runtime with `Class.getAnnotation()`. The new `@PoolDefinition` is a record processed at compile time:

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

The APT processor reads the record's fields and emits a JSON Schema to `META-INF/pool-definitions/code-reviewer.schema.json`. An IDE that understands JSON Schema can now autocomplete pool YAML — field names, types, legal values — without any manual schema authoring. The schema is always in sync with the code because it's generated from the code.

This is the pattern the platform's `StepPluginProcessor` already uses for step actions. Pool definitions now follow the same path: declare a record, get a schema.

## Build-time discovery replaces runtime reflection

Runtime reflection for configuration discovery is a solved problem in the JVM world — and the solution is to not do it. CDI, Spring, and Quarkus all moved to build-time processing years ago. The old `AgentPoolAnnotationScanner` was a throwback: scanning arbitrary classes for annotations at runtime, with no way for the framework to know what pool definitions exist until the scanning code actually runs.

The new `PoolDefinitionSource` reads `META-INF/pool-definitions/*.json` manifests from the classpath — exactly the same pattern as `AptPluginSource` for step plugins. The manifests are generated at compile time by the APT processor. Discovery is deterministic: what's on the classpath is what's discovered. No reflection, no class loading surprises, no GraalVM reflection configuration needed.

## The real value is consistency

None of these three things is individually hard to build from scratch. The old code worked. What platform alignment buys is that every YAML-configured component in the ecosystem — steps, pools, whatever comes next — uses the same typed parameter model, the same validation pipeline, and the same build-time discovery pattern. A developer who understands how step plugins work already understands how pool definitions work. The error messages look the same. The schema generation works the same. The classpath scanning works the same.

That's what "platform alignment" actually means. Not abstract consistency for its own sake, but the concrete elimination of bespoke parsing, validation, and discovery code that each new component would otherwise reinvent.
