# Pool YAML Platform Alignment Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #235 — epic: align agent pool canonical layer with platform YAML language
**Issue group:** #236, #237

**Goal:** Replace the agent pool YAML parser's manual extraction and the annotation scanner's runtime reflection with platform-standard typed validation (`StepDefinition`/`StepValidator`) and build-time annotation processing (`@PoolDefinition` record + APT).

**Architecture:** Two independent migrations sharing the same `AgentPoolDefinition` model. #236 replaces the YAML parser's internals with `StepDefinition` + `StepValidator` (Jackson still handles YAML→Map). #237 replaces `@PooledAgent`/`@AgentPool` annotations with a single `@PoolDefinition` record processed at compile time by a new `PoolDefinitionProcessor`, generating JSON Schema and classpath-discovery manifests.

**Tech Stack:** Java 21, casehub-platform-yaml-core (StepDefinition, StepValidator), casehub-platform-yaml-plugin-api (@Required, @Optional), javax.annotation.processing (APT)

## Global Constraints

- All files in `claudony-casehub` module (`casehub/src/main/java/io/casehub/claudony/casehub/fleet/`)
- Tests in `casehub/src/test/java/io/casehub/claudony/casehub/fleet/`
- `AgentPoolDefinition`, its builder DSL, and `AgentPoolDefinitionRegistry` are unchanged
- `AgentPoolConfigBridge` is unchanged (out of scope per #235)
- YAML surface format is unchanged — only internal parsing changes
- New dependencies use `${casehub.version}` (0.2-SNAPSHOT)
- Build: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl claudony-casehub`

---

## Batch 1: YAML Parser Migration (#236)

### Task 1: Add platform yaml-core dependency and define pool StepDefinition

**Files:**
- Modify: `pom.xml` (root — add yaml-core to dependencyManagement)
- Modify: `casehub/pom.xml` (add yaml-core dependency)
- Create: `casehub/src/main/java/io/casehub/claudony/casehub/fleet/AgentPoolSchema.java`
- Test: `casehub/src/test/java/io/casehub/claudony/casehub/fleet/AgentPoolSchemaTest.java`

**Interfaces:**
- Produces: `AgentPoolSchema.DEFINITION` — static `StepDefinition` describing the pool parameter schema. `AgentPoolSchema.flatten(String name, Map<String, Object> config)` — flattens a nested pool map entry into a validated flat map.

- [ ] **Step 1: Write the failing test for AgentPoolSchema**

```java
package io.casehub.claudony.casehub.fleet;

import io.casehub.yaml.core.step.StepParameterType;
import org.junit.jupiter.api.Test;

import java.util.Map;

import static org.assertj.core.api.Assertions.assertThat;

class AgentPoolSchemaTest {

    @Test
    void definitionHasExpectedInputs() {
        var def = AgentPoolSchema.DEFINITION;
        assertThat(def.name()).isEqualTo("agent-pool");
        assertThat(def.inputs()).containsKey("working-dir");
        assertThat(def.inputs()).containsKey("policy");
        assertThat(def.inputs()).containsKey("command");
        assertThat(def.inputs()).containsKey("pool.min-active");
        assertThat(def.inputs()).containsKey("pool.max-active");
        assertThat(def.inputs()).containsKey("pool.eviction");
    }

    @Test
    void policyHasAllowedValues() {
        var policy = AgentPoolSchema.DEFINITION.inputs().get("policy");
        assertThat(policy.type()).isEqualTo(StepParameterType.STRING);
        assertThat(policy.required()).isFalse();
        assertThat(policy.allowedValues()).containsExactlyInAnyOrder(
                "EXCLUSIVE", "SHARED_READ", "BRANCH_ISOLATED");
    }

    @Test
    void evictionHasAllowedValues() {
        var eviction = AgentPoolSchema.DEFINITION.inputs().get("pool.eviction");
        assertThat(eviction.type()).isEqualTo(StepParameterType.STRING);
        assertThat(eviction.allowedValues()).containsExactlyInAnyOrder(
                "MEMORY_WEIGHTED", "LRU");
    }

    @Test
    void integerFieldsHaveDefaults() {
        var minActive = AgentPoolSchema.DEFINITION.inputs().get("pool.min-active");
        assertThat(minActive.type()).isEqualTo(StepParameterType.INTEGER);
        assertThat(minActive.defaultValue()).isEqualTo("0");

        var maxActive = AgentPoolSchema.DEFINITION.inputs().get("pool.max-active");
        assertThat(maxActive.type()).isEqualTo(StepParameterType.INTEGER);
        assertThat(maxActive.defaultValue()).isEqualTo("10");
    }

    @Test
    void flattenMergesPoolSubMap() {
        var config = Map.<String, Object>of(
                "working-dir", "/workspace",
                "pool", Map.of("min-active", 2, "max-active", 5));

        var flat = AgentPoolSchema.flatten(config);

        assertThat(flat).containsEntry("working-dir", "/workspace");
        assertThat(flat).containsEntry("pool.min-active", 2);
        assertThat(flat).containsEntry("pool.max-active", 5);
        assertThat(flat).doesNotContainKey("pool");
    }

    @Test
    void flattenWithNoPoolSubMap() {
        var config = Map.<String, Object>of("command", "claude");
        var flat = AgentPoolSchema.flatten(config);
        assertThat(flat).containsEntry("command", "claude");
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl claudony-casehub -Dtest=AgentPoolSchemaTest`
Expected: FAIL — `AgentPoolSchema` does not exist

- [ ] **Step 3: Add yaml-core to dependency management and module pom**

Add to `pom.xml` root `<dependencyManagement>`:

```xml
<dependency>
    <groupId>io.casehub</groupId>
    <artifactId>casehub-platform-yaml-core</artifactId>
    <version>${casehub.version}</version>
</dependency>
```

Add to `casehub/pom.xml` `<dependencies>`:

```xml
<!-- Platform YAML core — StepDefinition, StepValidator, StepParameterType -->
<dependency>
    <groupId>io.casehub</groupId>
    <artifactId>casehub-platform-yaml-core</artifactId>
</dependency>
```

- [ ] **Step 4: Implement AgentPoolSchema**

```java
package io.casehub.claudony.casehub.fleet;

import io.casehub.yaml.core.step.StepDefinition;
import io.casehub.yaml.core.step.StepParameter;
import io.casehub.yaml.core.step.StepParameterType;

import java.util.Arrays;
import java.util.LinkedHashMap;
import java.util.Map;

public final class AgentPoolSchema {

    private AgentPoolSchema() {}

    public static final StepDefinition DEFINITION;

    static {
        var inputs = new LinkedHashMap<String, StepParameter>();

        inputs.put("working-dir", new StepParameter(
                StepParameterType.STRING, false, null, null, null, "Default working directory"));

        inputs.put("policy", new StepParameter(
                StepParameterType.STRING, false, "EXCLUSIVE",
                Arrays.stream(WorkingDirPolicy.values()).map(Enum::name).toList(),
                null, "Working directory concurrency policy"));

        inputs.put("command", new StepParameter(
                StepParameterType.STRING, false, null, null, null, "CLI command to run"));

        inputs.put("pool.min-active", new StepParameter(
                StepParameterType.INTEGER, false, "0", null, null, "Minimum pre-warmed sessions"));

        inputs.put("pool.max-active", new StepParameter(
                StepParameterType.INTEGER, false, "10", null, null, "Maximum concurrent active sessions"));

        inputs.put("pool.eviction", new StepParameter(
                StepParameterType.STRING, false, "MEMORY_WEIGHTED",
                Arrays.stream(EvictionStrategy.values()).map(Enum::name).toList(),
                null, "Eviction strategy when pool is at capacity"));

        DEFINITION = new StepDefinition("agent-pool", "Agent pool definition", inputs, Map.of(), null);
    }

    @SuppressWarnings("unchecked")
    public static Map<String, Object> flatten(Map<String, Object> config) {
        var flat = new LinkedHashMap<String, Object>();
        for (var entry : config.entrySet()) {
            if ("pool".equals(entry.getKey()) && entry.getValue() instanceof Map<?, ?> poolMap) {
                for (var poolEntry : ((Map<String, Object>) poolMap).entrySet()) {
                    flat.put("pool." + poolEntry.getKey(), poolEntry.getValue());
                }
            } else {
                flat.put(entry.getKey(), entry.getValue());
            }
        }
        return flat;
    }
}
```

- [ ] **Step 5: Run tests to verify they pass**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl claudony-casehub -Dtest=AgentPoolSchemaTest`
Expected: PASS (all 5 tests)

- [ ] **Step 6: Commit**

```bash
git add pom.xml casehub/pom.xml casehub/src/main/java/io/casehub/claudony/casehub/fleet/AgentPoolSchema.java casehub/src/test/java/io/casehub/claudony/casehub/fleet/AgentPoolSchemaTest.java
git commit -m "feat(#236): define pool StepDefinition schema with typed parameters

Adds AgentPoolSchema with a static StepDefinition describing the pool
parameter schema using platform typed parameters. Adds yaml-core dependency.

Refs #236"
```

### Task 2: Migrate AgentPoolYamlParser to use StepValidator

**Files:**
- Modify: `casehub/src/main/java/io/casehub/claudony/casehub/fleet/AgentPoolYamlParser.java`
- Modify: `casehub/src/test/java/io/casehub/claudony/casehub/fleet/AgentPoolYamlParserTest.java`

**Interfaces:**
- Consumes: `AgentPoolSchema.DEFINITION`, `AgentPoolSchema.flatten(Map)` from Task 1
- Produces: Same public API as before — `parse(String yaml)` returns `List<AgentPoolDefinition>`, `parseInto(String yaml, AgentPoolDefinitionRegistry registry)` registers all

- [ ] **Step 1: Add validation error tests to AgentPoolYamlParserTest**

Append these tests to the existing test class:

```java
@Test
void invalidPolicyThrowsWithMessage() {
    var yaml = """
            agent-pools:
              worker:
                policy: INVALID_POLICY
            """;

    assertThatThrownBy(() -> parser.parse(yaml))
            .isInstanceOf(IllegalArgumentException.class)
            .hasMessageContaining("policy")
            .hasMessageContaining("INVALID_POLICY");
}

@Test
void invalidEvictionThrowsWithMessage() {
    var yaml = """
            agent-pools:
              worker:
                pool:
                  eviction: INVALID_EVICTION
            """;

    assertThatThrownBy(() -> parser.parse(yaml))
            .isInstanceOf(IllegalArgumentException.class)
            .hasMessageContaining("eviction")
            .hasMessageContaining("INVALID_EVICTION");
}

@Test
void wrongTypeForMinActiveThrowsWithMessage() {
    var yaml = """
            agent-pools:
              worker:
                pool:
                  min-active: not-a-number
            """;

    assertThatThrownBy(() -> parser.parse(yaml))
            .isInstanceOf(IllegalArgumentException.class)
            .hasMessageContaining("min-active");
}
```

- [ ] **Step 2: Run the new tests to verify they fail**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl claudony-casehub -Dtest=AgentPoolYamlParserTest#invalidPolicyThrowsWithMessage+invalidEvictionThrowsWithMessage+wrongTypeForMinActiveThrowsWithMessage`
Expected: FAIL — current parser throws `IllegalArgumentException` from `valueOf()` but without the structured messages

- [ ] **Step 3: Rewrite AgentPoolYamlParser to use StepValidator**

Replace the entire class body with:

```java
package io.casehub.claudony.casehub.fleet;

import com.fasterxml.jackson.core.type.TypeReference;
import com.fasterxml.jackson.databind.ObjectMapper;
import com.fasterxml.jackson.dataformat.yaml.YAMLFactory;
import io.casehub.yaml.core.step.StepValidator;

import java.io.IOException;
import java.io.UncheckedIOException;
import java.util.ArrayList;
import java.util.Collections;
import java.util.List;
import java.util.Map;

public class AgentPoolYamlParser {

    private static final ObjectMapper YAML_MAPPER = new ObjectMapper(new YAMLFactory());

    public List<AgentPoolDefinition> parse(String yaml) {
        if (yaml == null || yaml.isBlank()) return Collections.emptyList();

        Map<String, Object> root;
        try {
            root = YAML_MAPPER.readValue(yaml, new TypeReference<>() {});
        } catch (IOException e) {
            throw new UncheckedIOException("Failed to parse agent pool YAML", e);
        }

        if (root == null || !root.containsKey("agent-pools")) return Collections.emptyList();

        @SuppressWarnings("unchecked")
        var pools = (Map<String, Map<String, Object>>) root.get("agent-pools");
        if (pools == null) return Collections.emptyList();

        var results = new ArrayList<AgentPoolDefinition>();
        for (var entry : pools.entrySet()) {
            results.add(toDefinition(entry.getKey(), entry.getValue()));
        }
        return results;
    }

    public void parseInto(String yaml, AgentPoolDefinitionRegistry registry) {
        for (var definition : parse(yaml)) {
            registry.register(definition);
        }
    }

    private AgentPoolDefinition toDefinition(String name, Map<String, Object> config) {
        if (config == null) config = Map.of();

        var flat = AgentPoolSchema.flatten(config);
        var errors = StepValidator.validateStep(name, flat, AgentPoolSchema.DEFINITION);
        if (!errors.isEmpty()) {
            throw new IllegalArgumentException(
                    "Invalid agent pool definition '" + name + "': " + String.join("; ", errors));
        }

        var agentBuilder = AgentPoolDefinition.builder().agent(name);

        var workingDir = (String) flat.get("working-dir");
        if (workingDir != null) agentBuilder.workingDir(workingDir);

        var policy = (String) flat.get("policy");
        if (policy != null) {
            agentBuilder.policy(WorkingDirPolicy.valueOf(policy.toUpperCase().replace('-', '_')));
        }

        var command = (String) flat.get("command");
        if (command != null) agentBuilder.command(command);

        var poolBuilder = agentBuilder.pool();

        var minActive = flat.get("pool.min-active");
        if (minActive instanceof Number n) poolBuilder.minActive(n.intValue());

        var maxActive = flat.get("pool.max-active");
        if (maxActive instanceof Number n) poolBuilder.maxActive(n.intValue());

        var eviction = (String) flat.get("pool.eviction");
        if (eviction != null) {
            poolBuilder.eviction(EvictionStrategy.valueOf(eviction.toUpperCase().replace('-', '_')));
        }

        return poolBuilder.build();
    }
}
```

- [ ] **Step 4: Run all parser tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl claudony-casehub -Dtest=AgentPoolYamlParserTest`
Expected: PASS (all existing + 3 new tests). Verify the validation error tests produce structured messages.

- [ ] **Step 5: Run full module tests to check for regressions**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl claudony-casehub`
Expected: PASS — no regressions

- [ ] **Step 6: Commit**

```bash
git add casehub/src/main/java/io/casehub/claudony/casehub/fleet/AgentPoolYamlParser.java casehub/src/test/java/io/casehub/claudony/casehub/fleet/AgentPoolYamlParserTest.java
git commit -m "feat(#236): migrate AgentPoolYamlParser to StepValidator

Replaces manual stringValue/intValue/parsePolicy/parseEviction helpers
with StepValidator.validateStep() against the typed AgentPoolSchema.
Adds validation error tests for invalid policy, eviction, and type mismatches.

Closes #236"
```

---

## Batch 2: Annotation Scanner Migration (#237)

### Task 3: Create @PoolDefinition annotation and PoolDefinitionProcessor APT

**Files:**
- Modify: `pom.xml` (root — add yaml-plugin-api to dependencyManagement)
- Modify: `casehub/pom.xml` (add yaml-plugin-api dependency, configure APT processor)
- Create: `casehub/src/main/java/io/casehub/claudony/casehub/fleet/PoolDefinition.java`
- Create: `casehub/src/main/java/io/casehub/claudony/casehub/fleet/PoolDefinitionProcessor.java`
- Create: `casehub/src/main/java/io/casehub/claudony/casehub/fleet/PoolSchemaEmitter.java`
- Create: `casehub/src/main/java/io/casehub/claudony/casehub/fleet/PoolManifestEmitter.java`
- Create: `casehub/src/main/resources/META-INF/services/javax.annotation.processing.Processor`
- Test: `casehub/src/test/java/io/casehub/claudony/casehub/fleet/PoolDefinitionProcessorTest.java`

**Interfaces:**
- Consumes: `@Required`, `@Optional` from `io.casehub.yaml.plugin.api`
- Produces: `@PoolDefinition` annotation. APT generates `META-INF/pool-definitions/<name>.json` (manifest) and `META-INF/pool-definitions/<name>.schema.json` (JSON Schema) at compile time.

- [ ] **Step 1: Write the @PoolDefinition annotation**

```java
package io.casehub.claudony.casehub.fleet;

import java.lang.annotation.ElementType;
import java.lang.annotation.Retention;
import java.lang.annotation.RetentionPolicy;
import java.lang.annotation.Target;

@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
public @interface PoolDefinition {
    String value();
    String description() default "";
}
```

- [ ] **Step 2: Write the APT processor test**

```java
package io.casehub.claudony.casehub.fleet;

import org.junit.jupiter.api.Test;
import org.junit.jupiter.api.io.TempDir;

import javax.tools.JavaCompiler;
import javax.tools.JavaFileObject;
import javax.tools.StandardJavaFileManager;
import javax.tools.ToolProvider;
import java.io.File;
import java.io.IOException;
import java.nio.file.Files;
import java.nio.file.Path;
import java.util.List;

import static org.assertj.core.api.Assertions.assertThat;

class PoolDefinitionProcessorTest {

    @TempDir
    Path tempDir;

    @Test
    void generatesSchemaAndManifest() throws Exception {
        var source = """
                package test;
                
                import io.casehub.claudony.casehub.fleet.PoolDefinition;
                import io.casehub.yaml.plugin.api.Required;
                import io.casehub.yaml.plugin.api.Optional;
                import io.casehub.claudony.casehub.fleet.WorkingDirPolicy;
                import io.casehub.claudony.casehub.fleet.EvictionStrategy;
                
                @PoolDefinition("code-reviewer")
                public record CodeReviewerPool(
                    @Optional String workingDir,
                    @Optional WorkingDirPolicy policy,
                    @Optional String command,
                    @Optional int minActive,
                    @Optional int maxActive,
                    @Optional EvictionStrategy eviction
                ) {}
                """;

        var outputDir = tempDir.resolve("classes");
        Files.createDirectories(outputDir);
        compile(source, "test/CodeReviewerPool.java", outputDir);

        var manifest = outputDir.resolve("META-INF/pool-definitions/code-reviewer.json");
        assertThat(manifest).exists();
        var manifestContent = Files.readString(manifest);
        assertThat(manifestContent).contains("\"name\": \"code-reviewer\"");
        assertThat(manifestContent).contains("\"recordClass\": \"test.CodeReviewerPool\"");

        var schema = outputDir.resolve("META-INF/pool-definitions/code-reviewer.schema.json");
        assertThat(schema).exists();
        var schemaContent = Files.readString(schema);
        assertThat(schemaContent).contains("\"working-dir\"");
        assertThat(schemaContent).contains("\"min-active\"");
        assertThat(schemaContent).contains("\"type\": \"integer\"");
        assertThat(schemaContent).contains("\"type\": \"string\"");
    }

    @Test
    void rejectsNonRecord() throws Exception {
        var source = """
                package test;
                
                import io.casehub.claudony.casehub.fleet.PoolDefinition;
                
                @PoolDefinition("bad")
                public class NotARecord {}
                """;

        var outputDir = tempDir.resolve("classes");
        Files.createDirectories(outputDir);

        var diagnostics = new java.util.ArrayList<javax.tools.Diagnostic<? extends JavaFileObject>>();
        compileWithDiagnostics(source, "test/NotARecord.java", outputDir, diagnostics);

        assertThat(diagnostics).anyMatch(d ->
                d.getMessage(null).contains("must be applied to a record"));
    }

    private void compile(String source, String fileName, Path outputDir) throws IOException {
        var sourceFile = tempDir.resolve(fileName);
        Files.createDirectories(sourceFile.getParent());
        Files.writeString(sourceFile, source);

        JavaCompiler compiler = ToolProvider.getSystemJavaCompiler();
        try (StandardJavaFileManager fm = compiler.getStandardFileManager(null, null, null)) {
            var compilationUnits = fm.getJavaFileObjects(sourceFile.toFile());
            var options = List.of("-d", outputDir.toString(),
                    "-classpath", System.getProperty("java.class.path"),
                    "-proc:only",
                    "-processor", "io.casehub.claudony.casehub.fleet.PoolDefinitionProcessor");
            var task = compiler.getTask(null, fm, null, options, null, compilationUnits);
            assertThat(task.call()).isTrue();
        }
    }

    private void compileWithDiagnostics(String source, String fileName, Path outputDir,
                                         List<javax.tools.Diagnostic<? extends JavaFileObject>> diagnostics) throws IOException {
        var sourceFile = tempDir.resolve(fileName);
        Files.createDirectories(sourceFile.getParent());
        Files.writeString(sourceFile, source);

        JavaCompiler compiler = ToolProvider.getSystemJavaCompiler();
        try (StandardJavaFileManager fm = compiler.getStandardFileManager(null, null, null)) {
            var listener = new javax.tools.DiagnosticCollector<JavaFileObject>();
            var compilationUnits = fm.getJavaFileObjects(sourceFile.toFile());
            var options = List.of("-d", outputDir.toString(),
                    "-classpath", System.getProperty("java.class.path"),
                    "-proc:only",
                    "-processor", "io.casehub.claudony.casehub.fleet.PoolDefinitionProcessor");
            var task = compiler.getTask(null, fm, listener, options, null, compilationUnits);
            task.call();
            diagnostics.addAll(listener.getDiagnostics());
        }
    }
}
```

- [ ] **Step 3: Run test to verify it fails**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl claudony-casehub -Dtest=PoolDefinitionProcessorTest`
Expected: FAIL — PoolDefinitionProcessor does not exist

- [ ] **Step 4: Add yaml-plugin-api dependency**

Add to `pom.xml` root `<dependencyManagement>`:

```xml
<dependency>
    <groupId>io.casehub</groupId>
    <artifactId>casehub-platform-yaml-plugin-api</artifactId>
    <version>${casehub.version}</version>
</dependency>
```

Add to `casehub/pom.xml` `<dependencies>`:

```xml
<!-- Platform YAML plugin API — @Required, @Optional annotations -->
<dependency>
    <groupId>io.casehub</groupId>
    <artifactId>casehub-platform-yaml-plugin-api</artifactId>
</dependency>
```

- [ ] **Step 5: Implement PoolSchemaEmitter**

```java
package io.casehub.claudony.casehub.fleet;

import javax.annotation.processing.Filer;
import javax.lang.model.element.RecordComponentElement;
import javax.tools.FileObject;
import javax.tools.StandardLocation;
import java.io.IOException;
import java.io.PrintWriter;
import java.util.List;

final class PoolSchemaEmitter {

    void emit(String name, List<RecordComponentElement> fields, Filer filer) throws IOException {
        FileObject file = filer.createResource(StandardLocation.CLASS_OUTPUT, "",
                "META-INF/pool-definitions/" + name + ".schema.json");

        try (PrintWriter w = new PrintWriter(file.openWriter())) {
            w.println("{");
            w.println("  \"$schema\": \"https://json-schema.org/draft/2020-12/schema\",");
            w.println("  \"type\": \"object\",");
            w.println("  \"properties\": {");

            for (int i = 0; i < fields.size(); i++) {
                RecordComponentElement field = fields.get(i);
                String yamlKey = toKebabCase(field.getSimpleName().toString());
                String jsonType = toJsonType(field.asType().toString());
                w.print("    \"" + yamlKey + "\": { \"type\": \"" + jsonType + "\" }");
                if (i < fields.size() - 1) w.print(",");
                w.println();
            }

            w.println("  },");
            w.println("  \"required\": [],");
            w.println("  \"additionalProperties\": false");
            w.println("}");
        }
    }

    static String toKebabCase(String camelCase) {
        StringBuilder sb = new StringBuilder();
        for (int i = 0; i < camelCase.length(); i++) {
            char c = camelCase.charAt(i);
            if (Character.isUpperCase(c) && i > 0) {
                sb.append('-');
                sb.append(Character.toLowerCase(c));
            } else {
                sb.append(c);
            }
        }
        return sb.toString();
    }

    private String toJsonType(String javaType) {
        return switch (javaType) {
            case "java.lang.String" -> "string";
            case "int", "long", "java.lang.Integer", "java.lang.Long" -> "integer";
            case "double", "float", "java.lang.Double", "java.lang.Float" -> "number";
            case "boolean", "java.lang.Boolean" -> "boolean";
            default -> "string";
        };
    }
}
```

- [ ] **Step 6: Implement PoolManifestEmitter**

```java
package io.casehub.claudony.casehub.fleet;

import javax.annotation.processing.Filer;
import javax.lang.model.element.TypeElement;
import javax.tools.FileObject;
import javax.tools.StandardLocation;
import java.io.IOException;
import java.io.PrintWriter;

final class PoolManifestEmitter {

    void emit(String name, TypeElement pluginClass, Filer filer) throws IOException {
        FileObject file = filer.createResource(StandardLocation.CLASS_OUTPUT, "",
                "META-INF/pool-definitions/" + name + ".json");

        try (PrintWriter w = new PrintWriter(file.openWriter())) {
            w.println("{");
            w.println("  \"name\": \"" + name + "\",");
            w.println("  \"recordClass\": \"" + pluginClass.getQualifiedName() + "\",");
            w.println("  \"schemaResource\": \"META-INF/pool-definitions/" + name + ".schema.json\"");
            w.println("}");
        }
    }
}
```

- [ ] **Step 7: Implement PoolDefinitionProcessor**

```java
package io.casehub.claudony.casehub.fleet;

import javax.annotation.processing.AbstractProcessor;
import javax.annotation.processing.RoundEnvironment;
import javax.annotation.processing.SupportedAnnotationTypes;
import javax.lang.model.SourceVersion;
import javax.lang.model.element.Element;
import javax.lang.model.element.ElementKind;
import javax.lang.model.element.RecordComponentElement;
import javax.lang.model.element.TypeElement;
import javax.tools.Diagnostic;
import java.util.ArrayList;
import java.util.List;
import java.util.Set;

@SupportedAnnotationTypes("io.casehub.claudony.casehub.fleet.PoolDefinition")
public class PoolDefinitionProcessor extends AbstractProcessor {

    private boolean processed;

    @Override
    public SourceVersion getSupportedSourceVersion() {
        return SourceVersion.latestSupported();
    }

    @Override
    public boolean process(Set<? extends TypeElement> annotations, RoundEnvironment roundEnv) {
        if (processed || roundEnv.processingOver()) return false;
        processed = true;

        for (Element element : roundEnv.getElementsAnnotatedWith(PoolDefinition.class)) {
            if (element.getKind() != ElementKind.RECORD) {
                error(element, "@PoolDefinition must be applied to a record");
                continue;
            }
            TypeElement typeElement = (TypeElement) element;
            PoolDefinition annotation = typeElement.getAnnotation(PoolDefinition.class);
            generate(annotation.value(), typeElement);
        }
        return false;
    }

    private void generate(String name, TypeElement typeElement) {
        List<RecordComponentElement> fields = new ArrayList<>(typeElement.getRecordComponents());
        try {
            new PoolSchemaEmitter().emit(name, fields, processingEnv.getFiler());
            new PoolManifestEmitter().emit(name, typeElement, processingEnv.getFiler());
        } catch (Exception e) {
            error(typeElement,
                    "Code generation failed for @PoolDefinition '" + name + "': " + e.getMessage());
        }
    }

    private void error(Element element, String message) {
        processingEnv.getMessager().printMessage(Diagnostic.Kind.ERROR, message, element);
    }
}
```

- [ ] **Step 8: Register the APT processor**

Create `casehub/src/main/resources/META-INF/services/javax.annotation.processing.Processor`:

```
io.casehub.claudony.casehub.fleet.PoolDefinitionProcessor
```

- [ ] **Step 9: Run APT processor tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl claudony-casehub -Dtest=PoolDefinitionProcessorTest`
Expected: PASS (both tests)

- [ ] **Step 10: Commit**

```bash
git add casehub/src/main/java/io/casehub/claudony/casehub/fleet/PoolDefinition.java casehub/src/main/java/io/casehub/claudony/casehub/fleet/PoolDefinitionProcessor.java casehub/src/main/java/io/casehub/claudony/casehub/fleet/PoolSchemaEmitter.java casehub/src/main/java/io/casehub/claudony/casehub/fleet/PoolManifestEmitter.java casehub/src/main/resources/META-INF/services/javax.annotation.processing.Processor casehub/src/test/java/io/casehub/claudony/casehub/fleet/PoolDefinitionProcessorTest.java pom.xml casehub/pom.xml
git commit -m "feat(#237): @PoolDefinition annotation with APT processor

Adds @PoolDefinition annotation for records, with PoolDefinitionProcessor
generating JSON Schema and manifest to META-INF/pool-definitions/.

Refs #237"
```

### Task 4: Create PoolDefinitionSource and delete old annotations

**Files:**
- Create: `casehub/src/main/java/io/casehub/claudony/casehub/fleet/PoolDefinitionSource.java`
- Delete: `casehub/src/main/java/io/casehub/claudony/casehub/fleet/PooledAgent.java` (use `ide_refactor_safe_delete`)
- Delete: `casehub/src/main/java/io/casehub/claudony/casehub/fleet/AgentPool.java` (use `ide_refactor_safe_delete`)
- Delete: `casehub/src/main/java/io/casehub/claudony/casehub/fleet/AgentPoolAnnotationScanner.java` (use `ide_refactor_safe_delete`)
- Rewrite: `casehub/src/test/java/io/casehub/claudony/casehub/fleet/AgentPoolAnnotationScannerTest.java` → rename to `PoolDefinitionSourceTest.java`
- Test: `casehub/src/test/java/io/casehub/claudony/casehub/fleet/PoolDefinitionSourceTest.java`

**Interfaces:**
- Consumes: `META-INF/pool-definitions/*.json` manifests (from Task 3 APT output), `AgentPoolDefinitionRegistry` (unchanged)
- Produces: `PoolDefinitionSource.discover(ClassLoader cl, AgentPoolDefinitionRegistry registry)` — scans classpath and populates registry

- [ ] **Step 1: Write the PoolDefinitionSource test**

```java
package io.casehub.claudony.casehub.fleet;

import com.fasterxml.jackson.databind.ObjectMapper;
import org.junit.jupiter.api.Test;
import org.junit.jupiter.api.io.TempDir;

import java.io.ByteArrayInputStream;
import java.io.InputStream;
import java.net.URL;
import java.net.URLClassLoader;
import java.nio.file.Files;
import java.nio.file.Path;

import static org.assertj.core.api.Assertions.assertThat;

class PoolDefinitionSourceTest {

    @TempDir
    Path tempDir;

    @Test
    void discoversManifestAndRegisters() throws Exception {
        var manifestDir = tempDir.resolve("META-INF/pool-definitions");
        Files.createDirectories(manifestDir);

        Files.writeString(manifestDir.resolve("code-reviewer.json"), """
                {
                  "name": "code-reviewer",
                  "recordClass": "test.CodeReviewerPool",
                  "schemaResource": "META-INF/pool-definitions/code-reviewer.schema.json"
                }
                """);

        Files.writeString(manifestDir.resolve("code-reviewer.schema.json"), """
                {
                  "type": "object",
                  "properties": {
                    "working-dir": { "type": "string" },
                    "min-active": { "type": "integer" },
                    "max-active": { "type": "integer" }
                  }
                }
                """);

        var cl = new URLClassLoader(new URL[]{tempDir.toUri().toURL()});
        var registry = new AgentPoolDefinitionRegistry();
        var source = new PoolDefinitionSource(new ObjectMapper());
        source.discover(cl, registry);

        assertThat(registry.size()).isEqualTo(1);
        assertThat(registry.get("code-reviewer")).isPresent();
        var def = registry.get("code-reviewer").get();
        assertThat(def.agent().name()).isEqualTo("code-reviewer");
    }

    @Test
    void emptyClasspathRegistersNothing() {
        var cl = new URLClassLoader(new URL[]{});
        var registry = new AgentPoolDefinitionRegistry();
        var source = new PoolDefinitionSource(new ObjectMapper());
        source.discover(cl, registry);

        assertThat(registry.isEmpty()).isTrue();
    }

    @Test
    void multipleManifestsDiscovered() throws Exception {
        var manifestDir = tempDir.resolve("META-INF/pool-definitions");
        Files.createDirectories(manifestDir);

        for (var name : new String[]{"alpha", "beta"}) {
            Files.writeString(manifestDir.resolve(name + ".json"), """
                    { "name": "%s", "recordClass": "test.%sPool" }
                    """.formatted(name, name.substring(0, 1).toUpperCase() + name.substring(1)));
        }

        var cl = new URLClassLoader(new URL[]{tempDir.toUri().toURL()});
        var registry = new AgentPoolDefinitionRegistry();
        new PoolDefinitionSource(new ObjectMapper()).discover(cl, registry);

        assertThat(registry.size()).isEqualTo(2);
        assertThat(registry.get("alpha")).isPresent();
        assertThat(registry.get("beta")).isPresent();
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl claudony-casehub -Dtest=PoolDefinitionSourceTest`
Expected: FAIL — `PoolDefinitionSource` does not exist

- [ ] **Step 3: Implement PoolDefinitionSource**

```java
package io.casehub.claudony.casehub.fleet;

import com.fasterxml.jackson.databind.ObjectMapper;

import java.io.File;
import java.io.IOException;
import java.io.InputStream;
import java.net.JarURLConnection;
import java.net.URL;
import java.util.Enumeration;
import java.util.LinkedHashMap;
import java.util.Map;
import java.util.jar.JarEntry;
import java.util.jar.JarFile;

public class PoolDefinitionSource {

    private static final String MANIFEST_DIR = "META-INF/pool-definitions/";

    private final ObjectMapper objectMapper;

    public PoolDefinitionSource(ObjectMapper objectMapper) {
        this.objectMapper = objectMapper;
    }

    public void discover(ClassLoader cl, AgentPoolDefinitionRegistry registry) {
        try {
            Enumeration<URL> dirs = cl.getResources(MANIFEST_DIR);
            while (dirs.hasMoreElements()) {
                URL dirUrl = dirs.nextElement();
                scanDirectory(dirUrl, cl, registry);
            }
        } catch (IOException e) {
            throw new IllegalStateException("Failed to scan pool definition manifests", e);
        }
    }

    private void scanDirectory(URL dirUrl, ClassLoader cl, AgentPoolDefinitionRegistry registry)
            throws IOException {
        String protocol = dirUrl.getProtocol();
        if ("file".equals(protocol)) {
            scanFileDirectory(new File(dirUrl.getPath()), cl, registry);
        } else if ("jar".equals(protocol)) {
            scanJarDirectory(dirUrl, cl, registry);
        }
    }

    private void scanFileDirectory(File dir, ClassLoader cl, AgentPoolDefinitionRegistry registry)
            throws IOException {
        File[] files = dir.listFiles();
        if (files == null) return;
        for (File file : files) {
            String name = file.getName();
            if (name.endsWith(".json") && !name.endsWith(".schema.json")) {
                String poolName = name.substring(0, name.length() - ".json".length());
                loadAndRegister(poolName, cl, registry);
            }
        }
    }

    private void scanJarDirectory(URL dirUrl, ClassLoader cl, AgentPoolDefinitionRegistry registry)
            throws IOException {
        JarURLConnection jarConn = (JarURLConnection) dirUrl.openConnection();
        try (JarFile jarFile = jarConn.getJarFile()) {
            Enumeration<JarEntry> jarEntries = jarFile.entries();
            while (jarEntries.hasMoreElements()) {
                JarEntry jarEntry = jarEntries.nextElement();
                String entryName = jarEntry.getName();
                if (entryName.startsWith(MANIFEST_DIR)
                        && entryName.endsWith(".json")
                        && !entryName.endsWith(".schema.json")
                        && !jarEntry.isDirectory()) {
                    String fileName = entryName.substring(MANIFEST_DIR.length());
                    String poolName = fileName.substring(0, fileName.length() - ".json".length());
                    loadAndRegister(poolName, cl, registry);
                }
            }
        }
    }

    @SuppressWarnings("unchecked")
    private void loadAndRegister(String poolName, ClassLoader cl, AgentPoolDefinitionRegistry registry)
            throws IOException {
        String manifestPath = MANIFEST_DIR + poolName + ".json";
        try (InputStream stream = cl.getResourceAsStream(manifestPath)) {
            if (stream == null) return;
            Map<String, Object> manifest = objectMapper.readValue(stream, LinkedHashMap.class);
            String name = (String) manifest.get("name");
            registry.register(AgentPoolDefinition.builder().agent(name).build());
        }
    }
}
```

- [ ] **Step 4: Run PoolDefinitionSource tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl claudony-casehub -Dtest=PoolDefinitionSourceTest`
Expected: PASS (all 3 tests)

- [ ] **Step 5: Delete old annotations and scanner**

Use `ide_refactor_safe_delete` on:
- `io.casehub.claudony.casehub.fleet.PooledAgent`
- `io.casehub.claudony.casehub.fleet.AgentPool`
- `io.casehub.claudony.casehub.fleet.AgentPoolAnnotationScanner`

Delete the old test file:
- `casehub/src/test/java/io/casehub/claudony/casehub/fleet/AgentPoolAnnotationScannerTest.java`

- [ ] **Step 6: Run full module tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl claudony-casehub`
Expected: PASS — no regressions

- [ ] **Step 7: Run full project tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test`
Expected: PASS — no regressions across all modules

- [ ] **Step 8: Commit**

```bash
git add -A
git commit -m "feat(#237): replace @PooledAgent/@AgentPool with @PoolDefinition + classpath discovery

Adds PoolDefinitionSource for classpath scanning of APT-generated manifests.
Deletes @PooledAgent, @AgentPool, and AgentPoolAnnotationScanner.

Closes #237"
```

---

## References

- [2026-09-28-pool-yaml-platform-alignment-design.md] — design spec this plan implements
- [AgentPoolYamlParser.java:30] — current parser being migrated
- [AgentPoolAnnotationScanner.java:11] — current scanner being replaced
- [AgentPoolDefinition.java:21] — immutable model (unchanged)
- [StepDefinition.java (casehub-platform-yaml-core)] — platform typed parameter model
- [StepValidator.java (casehub-platform-yaml-core)] — platform validation
- [StepPluginProcessor.java (yaml-plugin-processor)] — APT pattern reference
- [AptPluginSource.java (yaml-step-runtime)] — classpath scanning pattern reference
- [GitHub #235] — epic
- [GitHub #236, #237] — child issues
