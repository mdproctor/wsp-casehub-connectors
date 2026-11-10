# HANDOFF — casehub-connectors

## Last Session

Completed #187 — VoicePlatform SPI epic closed. Added voice MCP tools to the graphql module via `ConnectorVoiceApi` with `@McpDomain("connectors/voice")`:

- `voice_speak` — TTS with smart routing via `RoutingVoicePlatformService` (voice ID, alias, or query params), base64 audio output, optional phoneme timing
- `voice_transcribe` — STT with base64 audio input, platform routing
- `voice_list_voices` — query available voices by gender/accent/language/style/capability/vendor
- `voice_status` — `VoiceDomainReportProvider` for runtime platform status via `connectors/voice_report` tool

Uses platform MCP annotations (`@DefaultValue`, `McpCapabilityException`, platform `QueryParam`) rather than JAX-RS — following the "good MCP tool design" principle. `ConnectorsModelEnricher` updated with voice platform count. 12 new tests (144 total in graphql module). Landed as 58fd5923 on main.

## Immediate Next Step

Epic #187 is fully closed. No remaining items — cloud providers and per-provider module split are in the "Future" section, not blocking.

Open related work:
- agency#447 — Remove residual cognitive code from agency → neocortex (open, independent)
- agency#448 — speech-demo: replace hand-rolled Vertex client with agent-config manifest (open, independent)

## Cross-Module

- agency#447 — Remove residual cognitive code from agency → neocortex (open, independent)
- agency#448 — speech-demo: replace hand-rolled Vertex client with agent-config manifest (open, independent)

## References

- `graphql/src/main/java/io/casehub/connectors/graphql/ConnectorVoiceApi.java` — voice MCP tools
- `graphql/src/main/java/io/casehub/connectors/graphql/VoiceDomainReportProvider.java` — voice status report
- `graphql/src/main/java/io/casehub/connectors/graphql/dto/VoiceSynthesisResult.java` — TTS result DTO
- `voice-spi/src/main/java/io/casehub/connectors/voice/spi/RoutingVoicePlatformService.java` — routing layer
- `wsp-casehub-connectors/specs/issue-187-voice-mcp-tools/` — design spec and decisions
