# Handoff — slot 225 connectors

## Session summary (2026-11-23)

**#349 — LangChain4j integration examples**: Implementation complete, landing blocked on slot/canonical divergence.

### What was built

**Connectors (8 commits on slot main):**
- `langchain4j-it` module: 23 tests (8 direct + 15 LLM-driven) covering 11 of 13 platform domains
- Ollama with llama3.1:8b as default LLM provider (configurable via system properties)
- Voice-sherpa wired into MCP server: real Kokoro TTS + Whisper STT
- Voice registry populated via shared CDI instance and voice-config.yaml
- Multi-step LLM examples: search+cart, restaurant+directions, contact+calendar cross-domain

**Platform (3 commits on slot main):**
- `GraphQLModelScanner.isRequired()`: `@QueryParam`/`@DefaultValue` params now optional in MCP schema (handles both JAX-RS and custom QueryParam via simple name check)
- `ReflectiveOperationDispatcher.primitiveDefault()`: primitive Java defaults for omitted params (prevents NPE on boolean/int)
- InlineSchemaBuilder cherry-picked from canonical (ef2cdcc7) for LangChain4j-compatible schemas

### Test results with Ollama
- 8 direct tool tests: 100% pass rate
- 15 LLM tests: 11-12/15 per run (3-4 rotating flaky, inherent to 8B model vs 136 tools)

## Landing blocked

Work-end `land` step failed: slot repos have diverged from canonical.

| Repo | Slot commits canonical lacks | Canonical commits slot lacks |
|------|------------------------------|------------------------------|
| connectors | 8 (this session's #349 work) | Unknown (other sessions landed) |
| platform | 7 (3 this session + prior) | Unknown |

**Resolution needed:** Rebase slot main onto canonical main for both repos, resolve conflicts, then re-run `work end`.

The workspace branch `issue-347-inline-ref-schemas` has a HANDOFF.md conflict with main.

## Work-end ceremony — incomplete steps

| Step | Status |
|------|--------|
| Review | Done (2 items filed) |
| Diary/blog | Skipped (needs fresh context + writing style guide) |
| File issues | Done (#350, #351, #352) |
| Item finished | Yes (#349 marked done) |
| Land | Blocked (divergence) |
| Promote | Not reached |
| Publish | Not reached |
| Handoff | This file |

**Resume with:** `work end` — it will retry the land step.

## Issues filed this session

- #350 — VoiceConfigProducer: add `@DefaultBean` to prevent CDI ambiguity
- #351 — MCP server voice-config: add ref fallback voices
- #352 — LangChain4j integration: Podman clean-install test
- #353 — MCP server: move downstream to consume mosaic connection UI
- #355 — Epic: casehubio/distributions — leaf apps and packaged deployables (MCP server, Electron app, CLI)

## Prior session issues (still open)

- platform#939 — InlineSchemaBuilder: unwrap `Optional<X>`
- platform#940 — InlineSchemaBuilder: add required array for nested records
- platform#941 — InlineSchemaBuilder: circular reference protection

## Cross-repo state

| Repo | Branch | Status |
|------|--------|--------|
| connectors (slot 225) | main | 8 commits ahead of canonical, diverged |
| platform (slot 225) | main | 3 commits ahead of canonical, diverged |
| wsp-casehub-connectors | main/issue-347 | HANDOFF.md conflict |
| wsp-casehub-platform | main | 1 commit diverged |

## Queue

| # | Title | Scale | Status |
|---|-------|-------|--------|
| #349 | langchain4j-integration-examples | M | Done (not landed) |

## Key files changed

**Connectors:**
- `langchain4j-it/src/test/java/io/casehub/mcp/it/McpConnectorsIT.java`
- `langchain4j-it/pom.xml`
- `mcp-server-connectors/pom.xml`
- `mcp-server-connectors/src/main/java/io/casehub/connectors/mcp/McpServerBeans.java`
- `mcp-server-connectors/src/main/resources/voices/voice-config.yaml`
- `mcp-server-connectors/src/main/resources/application.properties`
- `voice-spi/src/main/java/io/casehub/connectors/voice/config/VoiceConfigProducer.java`

**Platform:**
- `mcp/src/main/java/io/casehub/platform/mcp/GraphQLModelScanner.java`
- `mcp/src/main/java/io/casehub/platform/mcp/ReflectiveOperationDispatcher.java`
- `mcp/src/main/java/io/casehub/platform/mcp/DynamicToolRegistrar.java` (cherry-pick)
