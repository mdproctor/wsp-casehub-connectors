# HANDOFF — casehub-connectors

## Last Session

Completed #202 — Voice provider VoicePlatform implementations and manifest routing. Added `KokoroVoicePlatform` (TTS-only) and `WhisperVoicePlatform` (STT-only) as per-engine composition roots in voice-sherpa. Per-engine design enables mix-and-match: deployments can use different providers for TTS and STT independently. Renamed `seed-catalog.yaml` to `voice-config.yaml` (matching `agent-config.yaml` convention). Updated defaults to kokoro (TTS) and whisper (STT). Landed as 392e77ca on main.

Also updated epic #187 — moved cloud provider items (OpenAI, Google) and per-provider module split to "Future (not blocking)" section. Cloud providers are gated on credential resolution infrastructure.

## Immediate Next Step

Continue epic #187 — two items remain:

1. `work start 187` — **MCP tools** (`voice_transcribe`, `voice_speak`). Follow the pattern from existing MCP tools (`send_chat`, `list_channels` etc. in the `mcp` module). Wire to `RoutingVoicePlatformService`.

2. **GraphQL wiring** — `ConnectorVoiceApi`. Follow the `graphql` module's `GraphQLResolverProcessor` pattern used by other connector SPIs.

Both are wiring/exposure work — connecting existing voice capabilities to the platform's API surface. No new voice logic needed.

## Cross-Module

- agency#447 — Remove residual cognitive code from agency → neocortex (open, independent)
- agency#448 — speech-demo: replace hand-rolled Vertex client with agent-config manifest (open, independent)

## References

- `voice-sherpa/src/main/java/io/casehub/connectors/voice/sherpa/KokoroVoicePlatform.java` — TTS platform
- `voice-sherpa/src/main/java/io/casehub/connectors/voice/sherpa/WhisperVoicePlatform.java` — STT platform
- `voice-spi/src/main/resources/voices/voice-config.yaml` — default voice manifest
- `voice-spi/src/main/java/io/casehub/connectors/voice/spi/RoutingVoicePlatformService.java` — routing layer
- `docs/specs/issue-202-voice-provider-manifest-routing/` — design spec and decisions
- `docs/blog/2026-11-10-mdp01-voice-providers-that-compose.md` — blog entry
