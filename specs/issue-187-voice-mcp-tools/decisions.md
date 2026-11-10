# Voice MCP Tools — Decisions

## D1: Tool surface

**Choice:** Four voice-resolved tools with rich routing — `voice_transcribe`, `voice_speak`, `voice_list_voices`, `voice_status`
**Alternatives:**
- Minimal (transcribe + speak only) — no discovery, agent must know platform IDs
- Fine-grained REST-style (separate tools per capability sub-interface) — too many tools for agent UX
**Rationale:** LLM agents benefit from smart voice routing (aliases, quality scoring) and voice discovery. Four tools is the right granularity.
**Trade-offs:** Slightly more implementation surface than minimal, but all four tools are thin wrappers over existing SPI.
**Sources:** Existing ConnectorXxxApi patterns in connectors/graphql, voice-spi interfaces
**Exploration:** quick
**Status:** captured

## D2: Audio data handling

**Choice:** Base64 strings in DTOs — API methods accept/return `String audioBase64`, conversion happens in the API layer
**Alternatives:**
- Raw byte[] with @McpTransform — more indirection, requires transform SPI bean
**Rationale:** Aligns with MCP AudioContent spec (base64 + mimeType). Works naturally with the platform's existing `ToolResponse.success(mapper.writeValueAsString(result))` pipeline. Direct and transparent.
**Trade-offs:** Slightly more memory for base64 encoding vs raw bytes, but typical TTS/STT payloads are small.
**Sources:** MCP spec 2025-11-25 AudioContent, platform DynamicToolRegistrar dispatch chain
**Exploration:** deep-analysis
**Status:** captured

## D3: Voice resolution strategy

**Choice:** Smart routing via `RoutingVoicePlatformService` — `voice` param can be ID, alias, or platform name; falls back to defaults; optional query params for attribute-based selection
**Alternatives:**
- Platform + voiceId explicit params — agent must call voice_list_voices first, no alias or quality routing
**Rationale:** The whole point of `RoutingVoicePlatformService` is to abstract voice selection. The MCP tool should expose that abstraction.
**Trade-offs:** Slightly more complex resolution logic in the API method, but the routing service handles it.
**Sources:** RoutingVoicePlatformService, VoiceRegistry, VoiceQuery
**Exploration:** quick
**Status:** captured

## D4: MCP annotations

**Choice:** Use platform MCP annotations beyond basic REST mapping — `@DefaultValue`, `McpCapabilityException`, `@McpEnrich`, `DomainReportProvider`
**Alternatives:**
- Basic REST-style annotations only (@PlatformQuery, @RestPath, @QueryParam) — misses MCP-specific affordances
**Rationale:** User directive: "follow the designs for good MCP tool designs, more annotation have been done for this — don't just follow rest." The newer annotations provide better agent UX.
**Trade-offs:** None significant — annotations are additive and the dispatcher already supports them.
**Sources:** Platform mcp-api annotations (McpEnrich, McpCapabilityException, DefaultValue, DomainReportProvider)
**Exploration:** quick
**Status:** captured
