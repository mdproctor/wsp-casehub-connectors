# POJO and Collection Return Type Support — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** platform#994 — APT processor: POJO and List<T> return type support for @AgentService
**Issue group:** platform#994, platform#995

**Goal:** Extend the `AgentServiceProcessor` APT processor to generate proxy code that handles POJO `T` and collection (`List<T>`, `Set<T>`, `Collection<T>`) return types with provider-native JSON schema output formatting, and validate unsupported return types at compile time.

**Architecture:** The APT processor classifies blocking return types (String → text, collection → array JSON schema + parse, any other declared type → object JSON schema + parse). The generated proxy gains `PlatformSchemaGenerator` and `ObjectMapper` constructor parameters. Schemas are cached at construction time. A new `AgentResponseParseException` carries raw response text when JSON parsing fails. Compile-time validation rejects void, primitives, arrays, and raw collections with clear error messages.

**Tech Stack:** Java 21, javax.annotation.processing (APT), javax.lang.model (TypeMirror/DeclaredType), google compile-testing (tests), PlatformSchemaGenerator (victools jsonschema-generator), Jackson ObjectMapper

## Global Constraints

- Module: `agent-service-generator` in platform repo at `/Users/mdproctor/claude/casehub/slots/225/platform/agent-service-generator`
- Agent API module: `/Users/mdproctor/claude/casehub/slots/225/platform/agent-api`
- Java 21 source, Java 26 JVM
- APT processor has no runtime dependencies beyond `agent-api` — generated code (not the processor) depends on `schema-generator` and Jackson
- Tests use `com.google.testing.compile:compile-testing:0.21.0` with `Compiler.javac().withProcessors()`
- All existing tests must continue to pass unchanged

---

## Batch 1: Foundation — AgentResponseParseException + constructor expansion

### Task 1: AgentResponseParseException and expanded proxy constructor

**Files:**
- Create: `agent-api/src/main/java/io/casehub/platform/agent/AgentResponseParseException.java`
- Create: `agent-api/src/test/java/io/casehub/platform/agent/AgentResponseParseExceptionTest.java`
- Modify: `agent-service-generator/src/main/java/io/casehub/platform/agent/service/generator/AgentServiceProcessor.java:62-83` (imports + constructor generation)
- Modify: `agent-service-generator/src/test/java/io/casehub/platform/agent/service/generator/AgentServiceProcessorTest.java` (update existing tests for new constructor)

**Interfaces:**
- Produces: `AgentResponseParseException(String message, Throwable cause, String rawResponse)` — used by Task 2 for POJO/collection parse error wrapping
- Produces: Updated proxy constructor signature `(AgentProvider, AgentResolver, PlatformSchemaGenerator, ObjectMapper)` — used by all subsequent tasks

- [ ] **Step 1: Write failing test for AgentResponseParseException**

Create the test file:

```java
// agent-api/src/test/java/io/casehub/platform/agent/AgentResponseParseExceptionTest.java
package io.casehub.platform.agent;

import org.junit.jupiter.api.Test;
import static org.assertj.core.api.Assertions.assertThat;

class AgentResponseParseExceptionTest {

    @Test
    void carriesMessageCauseAndRawResponse() {
        var cause = new RuntimeException("bad json");
        var ex = new AgentResponseParseException("Failed to parse as Task", cause, "{malformed}");

        assertThat(ex.getMessage()).isEqualTo("Failed to parse as Task");
        assertThat(ex.getCause()).isSameAs(cause);
        assertThat(ex.rawResponse()).isEqualTo("{malformed}");
    }

    @Test
    void isUnchecked() {
        assertThat(new AgentResponseParseException("msg", null, "raw"))
                .isInstanceOf(RuntimeException.class);
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl agent-api -Dtest=AgentResponseParseExceptionTest -f /Users/mdproctor/claude/casehub/slots/225/platform/pom.xml`
Expected: FAIL — class does not exist

- [ ] **Step 3: Implement AgentResponseParseException**

```java
// agent-api/src/main/java/io/casehub/platform/agent/AgentResponseParseException.java
package io.casehub.platform.agent;

public class AgentResponseParseException extends RuntimeException {

    private final String rawResponse;

    public AgentResponseParseException(String message, Throwable cause, String rawResponse) {
        super(message, cause);
        this.rawResponse = rawResponse;
    }

    public String rawResponse() {
        return rawResponse;
    }
}
```

- [ ] **Step 4: Run test to verify it passes**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl agent-api -Dtest=AgentResponseParseExceptionTest -f /Users/mdproctor/claude/casehub/slots/225/platform/pom.xml`
Expected: PASS

- [ ] **Step 5: Write failing test for expanded proxy constructor**

Add to `AgentServiceProcessorTest.java`:

```java
@Test
void generatedProxyHasExpandedConstructor() throws IOException {
    JavaFileObject source = JavaFileObjects.forSourceString("test.SimpleAgent",
        """
        package test;
        import io.casehub.platform.agent.AgentService;
        import io.casehub.platform.agent.Query;

        @AgentService(model = "fast")
        public interface SimpleAgent {
            @Query("Do something")
            String doIt();
        }
        """);

    Compilation compilation = Compiler.javac()
        .withProcessors(new AgentServiceProcessor())
        .compile(source);

    assertThat(compilation).succeeded();
    String proxy = readGenerated(compilation, "test.SimpleAgentProxy");
    assertThat(proxy).contains("PlatformSchemaGenerator schemaGenerator");
    assertThat(proxy).contains("ObjectMapper objectMapper");
    assertThat(proxy).contains("this.schemaGenerator = schemaGenerator");
    assertThat(proxy).contains("this.objectMapper = objectMapper");
}
```

- [ ] **Step 6: Run test to verify it fails**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl agent-service-generator -Dtest=AgentServiceProcessorTest#generatedProxyHasExpandedConstructor -f /Users/mdproctor/claude/casehub/slots/225/platform/pom.xml`
Expected: FAIL — constructor does not contain new parameters

- [ ] **Step 7: Update AgentServiceProcessor — imports and constructor**

Modify `AgentServiceProcessor.java`:

In `generateProxy()`, after the existing imports (line 71), add:

```java
out.println("import io.casehub.schema.generator.PlatformSchemaGenerator;");
out.println("import com.fasterxml.jackson.databind.ObjectMapper;");
out.println("import io.casehub.platform.agent.AgentResponseParseException;");
```

Replace the field declarations and constructor (lines 76-83) with:

```java
out.println("    private final AgentProvider agentProvider;");
out.println("    private final AgentResolver agentResolver;");
out.println("    private final PlatformSchemaGenerator schemaGenerator;");
out.println("    private final ObjectMapper objectMapper;");
out.println();
out.println("    public " + proxyName + "(AgentProvider agentProvider, AgentResolver agentResolver, PlatformSchemaGenerator schemaGenerator, ObjectMapper objectMapper) {");
out.println("        this.agentProvider = agentProvider;");
out.println("        this.agentResolver = agentResolver;");
out.println("        this.schemaGenerator = schemaGenerator;");
out.println("        this.objectMapper = objectMapper;");
out.println("    }");
```

- [ ] **Step 8: Update existing tests for new constructor assertions**

The existing test `generatedProxyHasCorrectConstructor` asserts `AgentProvider agentProvider` and `AgentResolver agentResolver` — these still pass because the new constructor contains both. Verify all existing tests still pass.

- [ ] **Step 9: Run all processor tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl agent-service-generator -f /Users/mdproctor/claude/casehub/slots/225/platform/pom.xml`
Expected: ALL PASS (including the new test)

- [ ] **Step 10: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/slots/225/platform add agent-api/src/main/java/io/casehub/platform/agent/AgentResponseParseException.java agent-api/src/test/java/io/casehub/platform/agent/AgentResponseParseExceptionTest.java agent-service-generator/src/main/java/io/casehub/platform/agent/service/generator/AgentServiceProcessor.java agent-service-generator/src/test/java/io/casehub/platform/agent/service/generator/AgentServiceProcessorTest.java
git -C /Users/mdproctor/claude/casehub/slots/225/platform commit -m "feat(#994): AgentResponseParseException + expanded proxy constructor

Add AgentResponseParseException to agent-api for structured JSON parse
error reporting with raw response text. Expand generated proxy constructor
to accept PlatformSchemaGenerator and ObjectMapper for POJO/collection
return type support.

Refs casehubio/platform#994"
```

---

## Batch 2: POJO and collection return type code generation

### Task 2: POJO return type support

**Files:**
- Modify: `agent-service-generator/src/main/java/io/casehub/platform/agent/service/generator/AgentServiceProcessor.java` (type classification helpers + generateMethod POJO path + schema field generation)
- Modify: `agent-service-generator/src/test/java/io/casehub/platform/agent/service/generator/AgentServiceProcessorTest.java` (POJO test)

**Interfaces:**
- Consumes: `AgentResponseParseException` from Task 1, expanded constructor from Task 1
- Produces: `isString()` helper, `isCollection()` helper, POJO code generation path in `generateMethod()` — used by Task 3 for collection path

- [ ] **Step 1: Write failing test for POJO return type**

Add to `AgentServiceProcessorTest.java`:

```java
@Test
void generatesProxyForPojoReturnType() throws IOException {
    JavaFileObject taskType = JavaFileObjects.forSourceString("test.Task",
        """
        package test;
        public record Task(String title, String description) {}
        """);

    JavaFileObject source = JavaFileObjects.forSourceString("test.TaskAgent",
        """
        package test;
        import io.casehub.platform.agent.AgentService;
        import io.casehub.platform.agent.Query;
        import io.casehub.platform.agent.Param;

        @AgentService(model = "fast")
        public interface TaskAgent {
            @Query("Get task ${id}")
            Task getTask(@Param("id") String id);
        }
        """);

    Compilation compilation = Compiler.javac()
        .withProcessors(new AgentServiceProcessor())
        .compile(taskType, source);

    assertThat(compilation).succeeded();
    String proxy = readGenerated(compilation, "test.TaskAgentProxy");
    assertThat(proxy).contains("ResponseFormat.json(\"Task\"");
    assertThat(proxy).contains("config = config.withResponseFormat");
    assertThat(proxy).contains("objectMapper.readValue(result.text(), test.Task.class)");
    assertThat(proxy).contains("AgentResponseParseException");
    assertThat(proxy).contains("private final String taskSchema");
    assertThat(proxy).contains("schemaGenerator.generate(test.Task.class)");
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl agent-service-generator -Dtest=AgentServiceProcessorTest#generatesProxyForPojoReturnType -f /Users/mdproctor/claude/casehub/slots/225/platform/pom.xml`
Expected: FAIL — no ResponseFormat.json in generated code

- [ ] **Step 3: Implement type classification helpers**

Add to `AgentServiceProcessor.java` after the existing `isAgentSession` method:

```java
private boolean isString(TypeMirror type) {
    if (type instanceof DeclaredType dt) {
        String raw = dt.asElement().toString();
        return raw.equals("java.lang.String");
    }
    return false;
}

private boolean isCollection(TypeMirror type) {
    if (type instanceof DeclaredType dt) {
        String raw = dt.asElement().toString();
        return raw.equals("java.util.List")
            || raw.equals("java.util.Set")
            || raw.equals("java.util.Collection");
    }
    return false;
}

private TypeMirror collectionElementType(TypeMirror type) {
    if (type instanceof DeclaredType dt && !dt.getTypeArguments().isEmpty()) {
        return dt.getTypeArguments().get(0);
    }
    return null;
}

private String collectionKind(TypeMirror type) {
    if (type instanceof DeclaredType dt) {
        String raw = dt.asElement().toString();
        if (raw.equals("java.util.Set")) return "Set";
        if (raw.equals("java.util.Collection")) return "Collection";
    }
    return "List";
}
```

- [ ] **Step 4: Add schema field generation to generateProxy**

In `generateProxy()`, before the method loop (line 85), scan methods and generate schema cache fields:

```java
// Schema fields for POJO/collection return types
java.util.Set<String> schemaTypes = new java.util.LinkedHashSet<>();
for (Element enclosed : iface.getEnclosedElements()) {
    if (enclosed instanceof ExecutableElement m) {
        TypeMirror rt = m.getReturnType();
        if (!isMultiAgentEvent(rt) && !isAgentSession(rt) && !isString(rt)) {
            if (isCollection(rt)) {
                TypeMirror elementType = collectionElementType(rt);
                if (elementType != null) schemaTypes.add(elementType.toString());
            } else if (rt instanceof DeclaredType) {
                schemaTypes.add(rt.toString());
            }
        }
    }
}
for (String schemaType : schemaTypes) {
    String fieldName = schemaFieldName(schemaType);
    out.println("    private final String " + fieldName + ";");
}
if (!schemaTypes.isEmpty()) {
    out.println();
}
```

Add the schema field name helper:

```java
private static String schemaFieldName(String qualifiedType) {
    String simple = qualifiedType.contains(".") ? qualifiedType.substring(qualifiedType.lastIndexOf('.') + 1) : qualifiedType;
    return Character.toLowerCase(simple.charAt(0)) + simple.substring(1) + "Schema";
}
```

In the constructor body, after `this.objectMapper = objectMapper;`, generate schema initialisation:

```java
for (String schemaType : schemaTypes) {
    String fieldName = schemaFieldName(schemaType);
    out.println("        this." + fieldName + " = schemaGenerator.generate(" + schemaType + ".class).toString();");
}
```

- [ ] **Step 5: Implement POJO code generation path in generateMethod**

Replace the else branch in `generateMethod()` (lines 142-148) — the section after `if (isMultiAgentEvent(returnType))`:

```java
if (isMultiAgentEvent(returnType)) {
    out.println("        return agentProvider.invoke(config);");
} else {
    String timeoutExpr = timeout > 0
        ? "Duration.ofSeconds(" + timeout + ")"
        : "Duration.ofSeconds(60)";
    out.println("        AgentInvocationResult result = agentProvider.invokeBlocking(config, " + timeoutExpr + ");");

    if (isString(returnType)) {
        out.println("        return result.text();");
    } else if (isCollection(returnType)) {
        // Task 3 fills this in
        out.println("        return result.text();"); // placeholder
    } else {
        // POJO path
        String qualifiedType = returnType.toString();
        String simpleName = qualifiedType.contains(".") ? qualifiedType.substring(qualifiedType.lastIndexOf('.') + 1) : qualifiedType;
        String fieldName = schemaFieldName(qualifiedType);
        out.println("        config = config.withResponseFormat(ResponseFormat.json(\"" + simpleName + "\", " + fieldName + "));");
        // Re-invoke with format
        out.println("        result = agentProvider.invokeBlocking(config, " + timeoutExpr + ");");
        out.println("        try {");
        out.println("            return objectMapper.readValue(result.text(), " + qualifiedType + ".class);");
        out.println("        } catch (com.fasterxml.jackson.core.JsonProcessingException e) {");
        out.println("            throw new AgentResponseParseException(\"Failed to parse response as " + simpleName + "\", e, result.text());");
        out.println("        }");
    }
}
```

Wait — the config should have the response format BEFORE invoking. Fix the generated code so config is set before `invokeBlocking`:

```java
if (isMultiAgentEvent(returnType)) {
    out.println("        return agentProvider.invoke(config);");
} else {
    String timeoutExpr = timeout > 0
        ? "Duration.ofSeconds(" + timeout + ")"
        : "Duration.ofSeconds(60)";

    if (isString(returnType)) {
        out.println("        AgentInvocationResult result = agentProvider.invokeBlocking(config, " + timeoutExpr + ");");
        out.println("        return result.text();");
    } else if (isCollection(returnType)) {
        // Task 3 fills this in — for now, same as String
        out.println("        AgentInvocationResult result = agentProvider.invokeBlocking(config, " + timeoutExpr + ");");
        out.println("        return null;");
    } else {
        // POJO path
        String qualifiedType = returnType.toString();
        String simpleName = qualifiedType.contains(".") ? qualifiedType.substring(qualifiedType.lastIndexOf('.') + 1) : qualifiedType;
        String fieldName = schemaFieldName(qualifiedType);
        out.println("        config = config.withResponseFormat(ResponseFormat.json(\"" + simpleName + "\", " + fieldName + "));");
        out.println("        AgentInvocationResult result = agentProvider.invokeBlocking(config, " + timeoutExpr + ");");
        out.println("        try {");
        out.println("            return objectMapper.readValue(result.text(), " + qualifiedType + ".class);");
        out.println("        } catch (com.fasterxml.jackson.core.JsonProcessingException e) {");
        out.println("            throw new AgentResponseParseException(\"Failed to parse response as " + simpleName + "\", e, result.text());");
        out.println("        }");
    }
}
```

- [ ] **Step 6: Run test to verify it passes**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl agent-service-generator -Dtest=AgentServiceProcessorTest#generatesProxyForPojoReturnType -f /Users/mdproctor/claude/casehub/slots/225/platform/pom.xml`
Expected: PASS

- [ ] **Step 7: Run all processor tests to check no regressions**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl agent-service-generator -f /Users/mdproctor/claude/casehub/slots/225/platform/pom.xml`
Expected: ALL PASS

- [ ] **Step 8: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/slots/225/platform add agent-service-generator/
git -C /Users/mdproctor/claude/casehub/slots/225/platform commit -m "feat(#994): POJO return type support in APT processor

Generate ResponseFormat.json() with cached schema and Jackson
deserialization for non-String, non-collection declared return types.
Wraps JsonProcessingException in AgentResponseParseException.

Refs casehubio/platform#994"
```

### Task 3: Collection return type support (List, Set, Collection)

**Files:**
- Modify: `agent-service-generator/src/main/java/io/casehub/platform/agent/service/generator/AgentServiceProcessor.java` (collection path in generateMethod + array schema fields + TypeReference import)
- Modify: `agent-service-generator/src/test/java/io/casehub/platform/agent/service/generator/AgentServiceProcessorTest.java` (List, Set, Collection, mixed tests)

**Interfaces:**
- Consumes: `isCollection()`, `collectionElementType()`, `collectionKind()`, `schemaFieldName()` from Task 2; expanded constructor from Task 1
- Produces: Complete collection code generation — no downstream tasks depend on this

- [ ] **Step 1: Write failing test for List return type**

Add to `AgentServiceProcessorTest.java`:

```java
@Test
void generatesProxyForListReturnType() throws IOException {
    JavaFileObject taskType = JavaFileObjects.forSourceString("test.Task",
        """
        package test;
        public record Task(String title, String description) {}
        """);

    JavaFileObject source = JavaFileObjects.forSourceString("test.TaskListAgent",
        """
        package test;
        import io.casehub.platform.agent.AgentService;
        import io.casehub.platform.agent.Query;
        import java.util.List;

        @AgentService(model = "fast")
        public interface TaskListAgent {
            @Query("List all tasks")
            List<Task> listTasks();
        }
        """);

    Compilation compilation = Compiler.javac()
        .withProcessors(new AgentServiceProcessor())
        .compile(taskType, source);

    assertThat(compilation).succeeded();
    String proxy = readGenerated(compilation, "test.TaskListAgentProxy");
    assertThat(proxy).contains("ResponseFormat.json(\"TaskList\"");
    assertThat(proxy).contains("config = config.withResponseFormat");
    assertThat(proxy).contains("TypeReference<java.util.List<test.Task>>");
    assertThat(proxy).contains("AgentResponseParseException");
    assertThat(proxy).contains("private final String taskListSchema");
}
```

- [ ] **Step 2: Write failing test for Set return type**

```java
@Test
void generatesProxyForSetReturnType() throws IOException {
    JavaFileObject taskType = JavaFileObjects.forSourceString("test.Task",
        """
        package test;
        public record Task(String title, String description) {}
        """);

    JavaFileObject source = JavaFileObjects.forSourceString("test.TaskSetAgent",
        """
        package test;
        import io.casehub.platform.agent.AgentService;
        import io.casehub.platform.agent.Query;
        import java.util.Set;

        @AgentService(model = "fast")
        public interface TaskSetAgent {
            @Query("Get unique tasks")
            Set<Task> uniqueTasks();
        }
        """);

    Compilation compilation = Compiler.javac()
        .withProcessors(new AgentServiceProcessor())
        .compile(taskType, source);

    assertThat(compilation).succeeded();
    String proxy = readGenerated(compilation, "test.TaskSetAgentProxy");
    assertThat(proxy).contains("new java.util.LinkedHashSet<>(");
    assertThat(proxy).contains("TypeReference<java.util.List<test.Task>>");
}
```

- [ ] **Step 3: Write failing test for Collection return type**

```java
@Test
void generatesProxyForCollectionReturnType() throws IOException {
    JavaFileObject taskType = JavaFileObjects.forSourceString("test.Task",
        """
        package test;
        public record Task(String title, String description) {}
        """);

    JavaFileObject source = JavaFileObjects.forSourceString("test.TaskCollectionAgent",
        """
        package test;
        import io.casehub.platform.agent.AgentService;
        import io.casehub.platform.agent.Query;
        import java.util.Collection;

        @AgentService(model = "fast")
        public interface TaskCollectionAgent {
            @Query("Get all tasks")
            Collection<Task> allTasks();
        }
        """);

    Compilation compilation = Compiler.javac()
        .withProcessors(new AgentServiceProcessor())
        .compile(taskType, source);

    assertThat(compilation).succeeded();
    String proxy = readGenerated(compilation, "test.TaskCollectionAgentProxy");
    assertThat(proxy).contains("TypeReference<java.util.List<test.Task>>");
    assertThat(proxy).doesNotContain("LinkedHashSet");
}
```

- [ ] **Step 4: Run tests to verify they fail**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl agent-service-generator -Dtest="AgentServiceProcessorTest#generatesProxyForListReturnType+generatesProxyForSetReturnType+generatesProxyForCollectionReturnType" -f /Users/mdproctor/claude/casehub/slots/225/platform/pom.xml`
Expected: FAIL — collection path not implemented

- [ ] **Step 5: Implement collection code generation path**

In `AgentServiceProcessor.java`, add the `TypeReference` import in `generateProxy()` (after the ObjectMapper import):

```java
out.println("import com.fasterxml.jackson.core.type.TypeReference;");
```

Replace the collection placeholder in `generateMethod()` with:

```java
} else if (isCollection(returnType)) {
    TypeMirror elementType = collectionElementType(returnType);
    String elementTypeName = elementType.toString();
    String elementSimple = elementTypeName.contains(".") ? elementTypeName.substring(elementTypeName.lastIndexOf('.') + 1) : elementTypeName;
    String kind = collectionKind(returnType);
    String schemaName = elementSimple + kind + "Schema";
    // Use element type's schema field — but wrap in array for the list schema field
    String elementFieldName = schemaFieldName(elementTypeName);
    out.println("        String " + schemaName + " = \"{\\\"type\\\":\\\"array\\\",\\\"items\\\":\" + " + elementFieldName + " + \"}\";");
    out.println("        config = config.withResponseFormat(ResponseFormat.json(\"" + elementSimple + kind + "\", " + schemaName + "));");
    out.println("        AgentInvocationResult result = agentProvider.invokeBlocking(config, " + timeoutExpr + ");");
    out.println("        try {");
    if (kind.equals("Set")) {
        out.println("            return new java.util.LinkedHashSet<>(objectMapper.readValue(result.text(), new TypeReference<java.util.List<" + elementTypeName + ">>(){}));");
    } else {
        out.println("            return objectMapper.readValue(result.text(), new TypeReference<java.util.List<" + elementTypeName + ">>(){});");
    }
    out.println("        } catch (com.fasterxml.jackson.core.JsonProcessingException e) {");
    out.println("            throw new AgentResponseParseException(\"Failed to parse response as " + kind + "<" + elementSimple + ">\", e, result.text());");
    out.println("        }");
}
```

- [ ] **Step 6: Run collection tests to verify they pass**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl agent-service-generator -Dtest="AgentServiceProcessorTest#generatesProxyForListReturnType+generatesProxyForSetReturnType+generatesProxyForCollectionReturnType" -f /Users/mdproctor/claude/casehub/slots/225/platform/pom.xml`
Expected: PASS

- [ ] **Step 7: Write and run test for mixed interface (String + POJO + List + streaming + session)**

Add to `AgentServiceProcessorTest.java`:

```java
@Test
void generatesProxyForMixedReturnTypes() throws IOException {
    JavaFileObject taskType = JavaFileObjects.forSourceString("test.Task",
        """
        package test;
        public record Task(String title, String description) {}
        """);

    JavaFileObject source = JavaFileObjects.forSourceString("test.FullAgent",
        """
        package test;
        import io.casehub.platform.agent.AgentService;
        import io.casehub.platform.agent.AgentEvent;
        import io.casehub.platform.agent.AgentSession;
        import io.casehub.platform.agent.Query;
        import io.smallrye.mutiny.Multi;
        import java.util.List;

        @AgentService(model = "fast")
        public interface FullAgent {
            @Query("Ask something")
            String ask();

            @Query("Get task")
            Task getTask();

            @Query("List tasks")
            List<Task> listTasks();

            @Query("Stream it")
            Multi<AgentEvent> stream();

            AgentSession session();
        }
        """);

    Compilation compilation = Compiler.javac()
        .withProcessors(new AgentServiceProcessor())
        .compile(taskType, source);

    assertThat(compilation).succeeded();
    String proxy = readGenerated(compilation, "test.FullAgentProxy");
    // String method — text path
    assertThat(proxy).contains("return result.text();");
    // POJO method — schema + parse
    assertThat(proxy).contains("objectMapper.readValue(result.text(), test.Task.class)");
    // List method — TypeReference
    assertThat(proxy).contains("TypeReference<java.util.List<test.Task>>");
    // Streaming method
    assertThat(proxy).contains("agentProvider.invoke(config)");
    // Session method
    assertThat(proxy).contains("agentProvider.openSession(init)");
    // Single schema field for Task (shared between POJO and List)
    assertThat(proxy).contains("private final String taskSchema");
}
```

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl agent-service-generator -f /Users/mdproctor/claude/casehub/slots/225/platform/pom.xml`
Expected: ALL PASS

- [ ] **Step 8: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/slots/225/platform add agent-service-generator/
git -C /Users/mdproctor/claude/casehub/slots/225/platform commit -m "feat(#994): collection return type support (List, Set, Collection)

Generate array-wrapped JSON schema and TypeReference-based Jackson
deserialization for List<T>, Set<T>, and Collection<T> return types.
Set wraps in LinkedHashSet; Collection deserializes as List.

Refs casehubio/platform#994"
```

---

## Batch 3: Compile-time validation (#995)

### Task 4: Return type validation with diagnostic errors

**Files:**
- Modify: `agent-service-generator/src/main/java/io/casehub/platform/agent/service/generator/AgentServiceProcessor.java` (validation in generateMethod + generateProxy)
- Modify: `agent-service-generator/src/test/java/io/casehub/platform/agent/service/generator/AgentServiceProcessorTest.java` (validation tests)

**Interfaces:**
- Consumes: Type classification helpers from Task 2
- Produces: Compile-time error diagnostics — no downstream tasks

- [ ] **Step 1: Write failing tests for rejected return types**

Add to `AgentServiceProcessorTest.java`:

```java
@Test
void rejectsVoidReturnType() {
    JavaFileObject source = JavaFileObjects.forSourceString("test.VoidAgent",
        """
        package test;
        import io.casehub.platform.agent.AgentService;
        import io.casehub.platform.agent.Query;

        @AgentService(model = "fast")
        public interface VoidAgent {
            @Query("Do something")
            void doIt();
        }
        """);

    Compilation compilation = Compiler.javac()
        .withProcessors(new AgentServiceProcessor())
        .compile(source);

    assertThat(compilation).failed();
    assertThat(compilation).hadErrorContaining("unsupported return type");
    assertThat(compilation).hadErrorContaining("void");
}

@Test
void rejectsPrimitiveReturnType() {
    JavaFileObject source = JavaFileObjects.forSourceString("test.PrimAgent",
        """
        package test;
        import io.casehub.platform.agent.AgentService;
        import io.casehub.platform.agent.Query;

        @AgentService(model = "fast")
        public interface PrimAgent {
            @Query("Count items")
            int count();
        }
        """);

    Compilation compilation = Compiler.javac()
        .withProcessors(new AgentServiceProcessor())
        .compile(source);

    assertThat(compilation).failed();
    assertThat(compilation).hadErrorContaining("unsupported return type");
    assertThat(compilation).hadErrorContaining("int");
}

@Test
void rejectsBoxedPrimitiveReturnType() {
    JavaFileObject source = JavaFileObjects.forSourceString("test.BoxedAgent",
        """
        package test;
        import io.casehub.platform.agent.AgentService;
        import io.casehub.platform.agent.Query;

        @AgentService(model = "fast")
        public interface BoxedAgent {
            @Query("Count items")
            Integer count();
        }
        """);

    Compilation compilation = Compiler.javac()
        .withProcessors(new AgentServiceProcessor())
        .compile(source);

    assertThat(compilation).failed();
    assertThat(compilation).hadErrorContaining("unsupported return type");
}

@Test
void rejectsArrayReturnType() {
    JavaFileObject taskType = JavaFileObjects.forSourceString("test.Task",
        """
        package test;
        public record Task(String title, String description) {}
        """);

    JavaFileObject source = JavaFileObjects.forSourceString("test.ArrayAgent",
        """
        package test;
        import io.casehub.platform.agent.AgentService;
        import io.casehub.platform.agent.Query;

        @AgentService(model = "fast")
        public interface ArrayAgent {
            @Query("Get tasks")
            Task[] getTasks();
        }
        """);

    Compilation compilation = Compiler.javac()
        .withProcessors(new AgentServiceProcessor())
        .compile(taskType, source);

    assertThat(compilation).failed();
    assertThat(compilation).hadErrorContaining("unsupported return type");
    assertThat(compilation).hadErrorContaining("List<Task>");
}

@Test
void rejectsRawCollectionReturnType() {
    JavaFileObject source = JavaFileObjects.forSourceString("test.RawListAgent",
        """
        package test;
        import io.casehub.platform.agent.AgentService;
        import io.casehub.platform.agent.Query;
        import java.util.List;

        @AgentService(model = "fast")
        @SuppressWarnings("rawtypes")
        public interface RawListAgent {
            @Query("Get items")
            List getItems();
        }
        """);

    Compilation compilation = Compiler.javac()
        .withProcessors(new AgentServiceProcessor())
        .compile(source);

    assertThat(compilation).failed();
    assertThat(compilation).hadErrorContaining("unsupported return type");
    assertThat(compilation).hadErrorContaining("type argument");
}
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl agent-service-generator -Dtest="AgentServiceProcessorTest#rejectsVoidReturnType+rejectsPrimitiveReturnType+rejectsBoxedPrimitiveReturnType+rejectsArrayReturnType+rejectsRawCollectionReturnType" -f /Users/mdproctor/claude/casehub/slots/225/platform/pom.xml`
Expected: FAIL — no validation, compilation succeeds

- [ ] **Step 3: Implement return type validation**

Add validation helper and boxed primitive set to `AgentServiceProcessor.java`:

```java
private static final java.util.Set<String> BOXED_PRIMITIVES = java.util.Set.of(
    "java.lang.Integer", "java.lang.Long", "java.lang.Double",
    "java.lang.Float", "java.lang.Boolean", "java.lang.Byte",
    "java.lang.Short", "java.lang.Character"
);

private static final String SUPPORTED_TYPES_MSG =
    "Supported: String, Multi<AgentEvent>, AgentSession, POJO classes, List<T>, Set<T>, Collection<T>.";
```

Add a `validateReturnType` method:

```java
private boolean validateReturnType(ExecutableElement method) {
    TypeMirror returnType = method.getReturnType();
    String methodName = method.getSimpleName().toString();
    String typeName = returnType.toString();

    // void
    if (returnType.getKind() == javax.lang.model.type.TypeKind.VOID) {
        processingEnv.getMessager().printMessage(Diagnostic.Kind.ERROR,
            "@AgentService method '" + methodName + "' has unsupported return type 'void'. " + SUPPORTED_TYPES_MSG, method);
        return false;
    }

    // primitives
    if (returnType.getKind().isPrimitive()) {
        processingEnv.getMessager().printMessage(Diagnostic.Kind.ERROR,
            "@AgentService method '" + methodName + "' has unsupported return type '" + typeName + "'. " + SUPPORTED_TYPES_MSG, method);
        return false;
    }

    // arrays
    if (returnType.getKind() == javax.lang.model.type.TypeKind.ARRAY) {
        String componentType = ((javax.lang.model.type.ArrayType) returnType).getComponentType().toString();
        String componentSimple = componentType.contains(".") ? componentType.substring(componentType.lastIndexOf('.') + 1) : componentType;
        processingEnv.getMessager().printMessage(Diagnostic.Kind.ERROR,
            "@AgentService method '" + methodName + "' has unsupported return type '" + typeName + "'. Use List<" + componentSimple + "> instead. " + SUPPORTED_TYPES_MSG, method);
        return false;
    }

    if (returnType instanceof DeclaredType dt) {
        String raw = dt.asElement().toString();

        // boxed primitives
        if (BOXED_PRIMITIVES.contains(raw)) {
            processingEnv.getMessager().printMessage(Diagnostic.Kind.ERROR,
                "@AgentService method '" + methodName + "' has unsupported return type '" + typeName + "'. Use a wrapper record instead. " + SUPPORTED_TYPES_MSG, method);
            return false;
        }

        // raw collection (no type arguments)
        if (isCollection(returnType) && dt.getTypeArguments().isEmpty()) {
            processingEnv.getMessager().printMessage(Diagnostic.Kind.ERROR,
                "@AgentService method '" + methodName + "' has unsupported return type '" + typeName + "'. Collection types require a type argument. " + SUPPORTED_TYPES_MSG, method);
            return false;
        }
    }

    return true;
}
```

Call `validateReturnType` at the start of `generateMethod()`, before any code generation:

```java
private void generateMethod(PrintWriter out, ExecutableElement method,
                             io.casehub.platform.agent.Query queryAnn,
                             String model, String descriptor, int timeout) {
    if (!validateReturnType(method)) return;
    // ... rest of method
```

Also add validation in `generateProxy()` for methods that have `@Query` and are not session methods — before the schema field scan. Add validation for void/@Query methods that bypass the generateMethod call too. Actually, since `generateMethod` is only called for `@Query`-annotated methods and non-session methods, we need to handle void there. But void methods with `@Query` will reach `generateMethod`, and the validation will catch them. Session methods without `@Query` are skipped by the `queryAnn == null` check and never reach validation, which is correct.

Also validate in the `generateProxy` schema-scan loop — if validation fails, skip the schema field generation for that type. Use a boolean flag from `validateReturnType` and track valid methods:

Actually simpler: run validation as a first pass and collect errors. If any errors, return early from `generateProxy()`. Add before the schema field scan:

```java
boolean hasErrors = false;
for (Element enclosed : iface.getEnclosedElements()) {
    if (enclosed instanceof ExecutableElement m) {
        if (!isAgentSession(m.getReturnType())) {
            var qAnn = m.getAnnotation(io.casehub.platform.agent.Query.class);
            if (qAnn != null && !validateReturnType(m)) {
                hasErrors = true;
            }
        }
    }
}
if (hasErrors) return;
```

Then remove the `validateReturnType` call from `generateMethod()` since it's now done in the first pass.

- [ ] **Step 4: Run validation tests to verify they pass**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl agent-service-generator -Dtest="AgentServiceProcessorTest#rejectsVoidReturnType+rejectsPrimitiveReturnType+rejectsBoxedPrimitiveReturnType+rejectsArrayReturnType+rejectsRawCollectionReturnType" -f /Users/mdproctor/claude/casehub/slots/225/platform/pom.xml`
Expected: PASS

- [ ] **Step 5: Run all processor tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl agent-service-generator -f /Users/mdproctor/claude/casehub/slots/225/platform/pom.xml`
Expected: ALL PASS

- [ ] **Step 6: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/slots/225/platform add agent-service-generator/
git -C /Users/mdproctor/claude/casehub/slots/225/platform commit -m "feat(#995): compile-time validation of unsupported return types

Reject void, primitives, boxed primitives, arrays, and raw collections
with clear diagnostic errors pointing to supported types. Array errors
suggest List<T> alternative.

Refs casehubio/platform#995"
```

---

## References

- [2026-11-26-pojo-list-return-types-design.md] — design spec this plan implements
- `AgentServiceProcessor.java` (platform/agent-service-generator/src/main/java) — APT processor being modified
- `AgentServiceProcessorTest.java` (platform/agent-service-generator/src/test/java) — existing compile-testing tests
- `ResponseFormat.java` (platform/agent-api) — `json(name, schema)` factory
- `AgentSessionConfig.java` (platform/agent-api) — `withResponseFormat()` wiring
- `PlatformSchemaGenerator.java` (platform/schema-generator) — runtime JSON schema generation
- platform#994 — POJO and List<T> return type support
- platform#995 — compile-time validation of unsupported return types
