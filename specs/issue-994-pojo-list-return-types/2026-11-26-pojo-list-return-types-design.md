# APT Processor: POJO and Collection Return Type Support

**Issues:** platform#994 (POJO/List return types), platform#995 (return type validation)
**Branch:** issue-994-pojo-list-return-types
**Module:** `agent-service-generator` (platform repo, slot 225)

## Problem

The `AgentServiceProcessor` APT processor generates proxy classes from `@AgentService` interfaces. Currently, all blocking methods generate `result.text()` regardless of return type. A method returning `Task` generates code that returns a `String` where `Task` is expected — a compile error.

## Design

### Return type classification

`generateMethod()` classifies the blocking return type into three categories:

| Return type | Category | Generated code |
|------------|----------|----------------|
| `String` | text | `result.text()` (unchanged) |
| `List<T>`, `Set<T>`, `Collection<T>` | collection | JSON schema + parse to collection |
| Any other declared type | pojo | JSON schema + parse to POJO |

The streaming (`Multi<AgentEvent>`) and session (`AgentSession`) paths are unchanged — they are detected before `generateMethod()` is called.

### Generated proxy changes

**Constructor signature** gains two parameters:

```java
public GreeterProxy(AgentProvider agentProvider, AgentResolver agentResolver,
                    PlatformSchemaGenerator schemaGenerator, ObjectMapper objectMapper) {
```

Both are constructor-injected so CDI wiring provides the application's configured instances.

**Schema caching:** For each POJO/collection return type in the interface, the proxy generates a private field that holds the pre-computed JSON schema string. The schema is computed once at construction time:

```java
private final String taskSchema;

public TaskAgentProxy(...) {
    // ...
    this.taskSchema = schemaGenerator.generate(Task.class).toString();
}
```

**POJO return path:**

```java
config = config.withResponseFormat(ResponseFormat.json("Task", taskSchema));
AgentInvocationResult result = agentProvider.invokeBlocking(config, timeout);
return objectMapper.readValue(result.text(), Task.class);
```

**Collection return path** (List, Set, Collection):

For `List<Task>`, the schema wraps the element schema in an array envelope. The generated code uses Jackson's `TypeReference` for type-safe deserialization:

```java
config = config.withResponseFormat(ResponseFormat.json("TaskList", taskListSchema));
AgentInvocationResult result = agentProvider.invokeBlocking(config, timeout);
return objectMapper.readValue(result.text(), new com.fasterxml.jackson.core.type.TypeReference<java.util.List<Task>>(){});
```

For `Set<T>` and `Collection<T>`, the generated code deserializes as `List<T>` (JSON arrays naturally map to lists) and wraps in the appropriate collection type:
- `Set<T>` → `new java.util.LinkedHashSet<>(objectMapper.readValue(...))`
- `Collection<T>` → deserialized directly as `List<T>` (List implements Collection)

**Array schema generation:** For collection types, the proxy generates an array-wrapping schema at construction:

```java
this.taskListSchema = "{\"type\":\"array\",\"items\":" + schemaGenerator.generate(Task.class).toString() + "}";
```

### Type detection in the APT processor

The processor uses `TypeMirror` and `DeclaredType` to classify return types:

1. Check `isMultiAgentEvent(returnType)` — streaming (existing)
2. Check `isAgentSession(returnType)` — session (existing, checked before `generateMethod`)
3. In `generateMethod()`:
   - `isString(returnType)` — text path
   - `isCollection(returnType)` → extract element type from first type argument → collection path
   - Otherwise → POJO path

`isCollection()` checks whether the erased type name is one of `java.util.List`, `java.util.Set`, or `java.util.Collection`.

Element type extraction: `((DeclaredType) returnType).getTypeArguments().get(0)` gives the `TypeMirror` for `T`. The processor converts this to a qualified class name for the generated code.

### Return type validation (#995)

Before generating code, `generateMethod()` validates the return type:

**Rejected types** (emit `Diagnostic.Kind.ERROR`):
- `void` — no return value to map
- Primitives (`int`, `boolean`, `char`, etc.) — not supported by JSON schema
- Boxed primitives (`Integer`, `Boolean`, etc.) — use a wrapper record
- Arrays (`Task[]`) — use `List<Task>` instead
- Raw collection types (`List` without type argument) — element type required
- `Multi` without `AgentEvent` type argument — only `Multi<AgentEvent>` is valid for streaming

**Error message format:**
```
@AgentService method 'decompose' has unsupported return type 'void'.
Supported: String, Multi<AgentEvent>, AgentSession, POJO classes, List<T>, Set<T>, Collection<T>.
```

### Dependency changes

`agent-service-generator/pom.xml` — no new compile dependencies. The processor itself only uses `javax.annotation.processing` and `javax.lang.model` APIs.

The **generated proxy code** references `PlatformSchemaGenerator` and `ObjectMapper`. These are dependencies of the consuming module (the one that declares `@AgentService` interfaces), not of the generator itself. The generator's pom description already documents this: "Supports blocking (String, T, List)".

### Import generation

The processor adds these imports to the generated proxy when POJO/collection return types are present:

```java
import io.casehub.schema.generator.PlatformSchemaGenerator;
import com.fasterxml.jackson.databind.ObjectMapper;
import com.fasterxml.jackson.core.type.TypeReference;  // only for collection returns
```

These imports are conditional — only emitted when the interface has methods with non-String blocking return types.

### Error handling

`objectMapper.readValue()` throws `JsonProcessingException` (checked). The generated code wraps it in an unchecked exception:

```java
try {
    return objectMapper.readValue(result.text(), Task.class);
} catch (com.fasterxml.jackson.core.JsonProcessingException e) {
    throw new io.casehub.platform.agent.AgentResponseParseException(
        "Failed to parse response as Task", e, result.text());
}
```

`AgentResponseParseException` is a new unchecked exception in `agent-api` that carries the raw response text for diagnostics.

### Test plan

1. **POJO return type** — interface with `@Query Task getTask(String id)`, verify generated proxy contains `ResponseFormat.json`, `objectMapper.readValue`, correct schema field
2. **List return type** — `List<Task> listTasks()`, verify array schema wrapping, TypeReference generation
3. **Set return type** — `Set<Task> uniqueTasks()`, verify LinkedHashSet wrapping
4. **Collection return type** — `Collection<Task> allTasks()`, verify List deserialization
5. **Mixed interface** — String + POJO + List + streaming + session methods coexist, verify correct code generation for each
6. **Validation: void** — verify `Diagnostic.Kind.ERROR` emitted
7. **Validation: primitive** — `int count()` rejected
8. **Validation: boxed primitive** — `Integer count()` rejected
9. **Validation: array** — `Task[] tasks()` rejected with guidance
10. **Validation: raw List** — `List tasks()` rejected
11. **Existing tests pass** — String, streaming, session paths unchanged

## References

- `AgentServiceProcessor.java` (platform/agent-service-generator) — the APT processor being modified
- `ResponseFormat.java` (platform/agent-api) — `json(name, schema)` factory
- `AgentSessionConfig.java` (platform/agent-api) — `withResponseFormat()` wiring
- `PlatformSchemaGenerator.java` (platform/schema-generator) — runtime JSON schema generation
- `AgentServiceProcessorTest.java` — existing compile-testing tests
- platform#994 — POJO and List<T> return type support
- platform#995 — compile-time validation of unsupported return types
