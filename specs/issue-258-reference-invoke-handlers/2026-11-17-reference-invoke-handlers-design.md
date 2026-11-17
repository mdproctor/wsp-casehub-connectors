# Reference InvokeHandler implementations for messaging extensions

**Issue:** casehubio/connectors#258
**Branch:** issue-258-reference-invoke-handlers
**Date:** 2026-11-17

## Problem

The InvokeHandler SPI dispatches composeExtension/* invoke activities, but the
connectors repo has no reference implementations. Consumers need working examples
to build from, and tests need lightweight canned handlers for contract verification.

## Solution

Four public InvokeHandler implementations in the `chat-teams` module under a
`ref` sub-package. Each returns hard-coded canned responses using the
ComposeAttachment builder (from #259). They follow the established ref
implementation pattern: in-memory, no external dependencies, usable in tests
and demos.

## Handlers

### RefSearchHandler — `composeExtension/query`

Returns 3 hard-coded search results as a `ComposeResult("list", ...)` with
thumbnail preview cards.

```java
public class RefSearchHandler implements InvokeHandler {
    public String name() { return "composeExtension/query"; }

    public InvokeResponse handle(InvokeRequest request) {
        return new InvokeResponse.ComposeResult("list", List.of(
            ComposeAttachment.adaptive(CARD_1).preview("Contoso Widget", "£29.99").build(),
            ComposeAttachment.adaptive(CARD_2).preview("Northwind Gadget", "£49.99").build(),
            ComposeAttachment.adaptive(CARD_3).preview("Adventure Gizmo", "£19.99").build()
        ));
    }
}
```

Each `CARD_N` is a minimal adaptive card JSON constant with a TextBlock showing
the product name.

### RefLinkUnfurlHandler — `composeExtension/queryLink`

Returns a single unfurl card as a `ComposeResult("list", ...)` without preview
(unfurl cards render inline, no thumbnail needed).

```java
public class RefLinkUnfurlHandler implements InvokeHandler {
    public String name() { return "composeExtension/queryLink"; }

    public InvokeResponse handle(InvokeRequest request) {
        return new InvokeResponse.ComposeResult("list", List.of(
            ComposeAttachment.adaptive(UNFURL_CARD).build()
        ));
    }
}
```

The unfurl card shows a TextBlock with "Link Preview" and an action button.

### RefFetchTaskHandler — `composeExtension/fetchTask`

Returns a `ComposeContinue` with a simple adaptive card form containing a text
input and a submit action.

```java
public class RefFetchTaskHandler implements InvokeHandler {
    public String name() { return "composeExtension/fetchTask"; }

    public InvokeResponse handle(InvokeRequest request) {
        return new InvokeResponse.ComposeContinue("Create Item", FORM_CARD);
    }
}
```

The form card has a TextBlock title, a Text.Input for "Title", and an
Action.Submit button.

### RefSubmitActionHandler — `composeExtension/submitAction`

Returns a `ComposeResult("list", ...)` with a confirmation card built from
ComposeAttachment.

```java
public class RefSubmitActionHandler implements InvokeHandler {
    public String name() { return "composeExtension/submitAction"; }

    public InvokeResponse handle(InvokeRequest request) {
        return new InvokeResponse.ComposeResult("list", List.of(
            ComposeAttachment.adaptive(CONFIRMATION_CARD)
                .preview("Item Created", "Action completed")
                .build()
        ));
    }
}
```

The confirmation card shows a TextBlock with "Item Created Successfully".

## Design decisions

- **D1 — Module placement:** Directly in `chat-teams` module. InvokeHandler is
  Teams-specific, not a cross-platform SPI. No need for a separate ref module.
  Handlers go in a `ref` sub-package.
- **D2 — Handler granularity:** One class per invoke name (4 total).
  InvokeHandler.name() returns a single string, so each handler maps to exactly
  one invoke name.
- **D3 — Canned data:** Hard-coded fixed responses regardless of input. Matches
  the ref implementation pattern. Purpose is demonstrating the handler contract
  and ComposeAttachment usage, not simulating real behaviour.

## Package structure

```
chat-teams/src/main/java/io/casehub/connectors/chat/teams/ref/
    RefSearchHandler.java
    RefLinkUnfurlHandler.java
    RefFetchTaskHandler.java
    RefSubmitActionHandler.java
```

Tests:
```
chat-teams/src/test/java/io/casehub/connectors/chat/teams/ref/
    RefSearchHandlerTest.java
    RefLinkUnfurlHandlerTest.java
    RefFetchTaskHandlerTest.java
    RefSubmitActionHandlerTest.java
```

## CDI integration

Each handler is annotated `@ApplicationScoped` for CDI discovery by
InvokeService. In production, consumers enable the ref handlers via a build
profile or exclude them — but the default is available, matching how chat-ref
and other ref implementations work.

Actually, ref implementations in this codebase are NOT `@ApplicationScoped` —
they're plain classes instantiated directly. `RefChatPlatform` has no CDI
annotations. Handlers should follow the same pattern: plain classes, no CDI
annotations, instantiated by consumers where needed.

## Testing

Each handler gets a unit test verifying:
1. Correct invoke name from `name()`
2. Response is the expected InvokeResponse variant
3. JSON output (via InvokeResponseSerializer) has the correct structure

A round-trip integration test verifies all 4 handlers can be collected into an
InvokeService and dispatched correctly.

## What doesn't change

- InvokeHandler, InvokeRequest, InvokeResponse — untouched
- InvokeService dispatch — unchanged
- ComposeAttachment — consumed, not modified
- No new module, no new dependencies

## References

- [chat-teams/src/main/java/io/casehub/connectors/chat/teams/InvokeHandler.java] — SPI interface
- [chat-teams/src/main/java/io/casehub/connectors/chat/teams/InvokeService.java] — dispatch by name
- [chat-teams/src/main/java/io/casehub/connectors/chat/teams/InvokeResponse.java] — response variants
- [chat-teams/src/main/java/io/casehub/connectors/chat/teams/ComposeAttachment.java] — builder from #259
- [chat-teams/src/main/java/io/casehub/connectors/chat/teams/InvokeResponseSerializer.java] — serialization
- [chat-ref/src/main/java/io/casehub/connectors/chat/ref/RefChatPlatform.java] — ref implementation pattern
- [GitHub #258] — issue: reference InvokeHandler implementations
- [GitHub #259] — ComposeAttachment builder (dependency)
