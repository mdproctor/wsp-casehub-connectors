---
layout: post
title: "The slash that slipped through — normalising MCP tool names"
date: 2026-11-20
entry_type: note
subtype: diary
projects: [casehubio/connectors, casehubio/platform]
tags: [mcp, naming-consistency, code-review, bug-fix]
---

# The slash that slipped through — normalising MCP tool names

Domain names in the platform use slashes as namespace separators: `connectors/voice`, `connectors/chat`, `neocortex/reasoning`. When these get turned into MCP tool names, the slashes need to become underscores — MCP tool names can't contain slashes. The flat-mode tool registration in `DynamicToolRegistrar` does this correctly:

```java
String toolName = domain.name().replace("/", "_") + "_" + op.name();
// connectors/chat + send → connectors_chat_send ✓
```

The report tool registration, added in the same class a few commits later, doesn't:

```java
String toolName = domain + "_report";
// connectors/voice → connectors/voice_report ✗
```

Claude caught this during the domain filtering test review — the test was asserting that `connectors/voice_report` should be null (excluded by the domain filter), which passed, but for the wrong reason: the tool name itself was malformed regardless of filtering.

## One bug, three locations

The fix is trivial — add `.replace("/", "_")` to the report tool name. But code review found the same bug in two other places:

**`activateDomain`** — the dispatch-mode path where a client calls `casehub_activate` to expand a domain into individual tools at runtime. Same pattern: `domain + "_" + op.name()` without the slash replacement. A client activating `connectors/voice` would get tools named `connectors/voice_speak` instead of `connectors_voice_speak`.

**`CaseHubToolCallbackProvider`** — the Spring Boot auto-configuration counterpart. This one was interesting because it already had `domain.replace("-", "_")` for hyphen normalisation, but missed the slash. The Spring and Quarkus paths share no code — they're independent implementations of the same registration logic, so the same kind of bug appears independently.

Three code paths, three fixes, all one-liners. The test uses a `test/nested` domain with a report provider and a query resolver, verifying that all three paths produce `test_nested_*` tool names, not `test/nested_*`.

## Why it matters

A slash in an MCP tool name isn't just an aesthetic problem. MCP clients parse tool names for routing, logging, and auto-completion. Some frameworks (Claude Desktop's tool approval dialog, for one) display the tool name in a UI context where a slash could be misread as a path separator. The tool might still work — the MCP protocol doesn't formally ban slashes in tool names — but it creates inconsistency that confuses both humans and LLMs doing tool selection.

The deeper point is about consistency surfaces. When you have three independent code paths that construct tool names — flat registration, report registration, dynamic activation — they need to agree on the normalisation rules. The flat path got it right because it was written first and the domain names it was handling already had slashes. The other two were written later, by which point the convention was established but not extracted into a shared method. Convention without code is just a hope.
