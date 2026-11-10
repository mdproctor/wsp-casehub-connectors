# Voice MCP Tools — Design

## Context

connectors#187 requires MCP tools for the VoicePlatform SPI: `voice_transcribe` (STT) and `voice_speak` (TTS), plus discovery tools. The voice SPI, routing service, and voice registry are already implemented. This spec covers the `ConnectorVoiceApi` class in the graphql module and supporting DTOs.

## Architecture

A single `ConnectorVoiceApi` class in `connectors/graphql`, annotated with `@McpDomain("connectors/voice")`, following the same pattern as other connector APIs (CalendarApi, DocumentApi, HealthApi). The class delegates to `RoutingVoicePlatformService` for voice resolution and `VoicePlatformService` for direct platform access.

The `ConnectorsModelEnricher` is updated to include voice platform count in state.

The graphql pom.xml is updated to depend on `casehub-connectors-voice-spi` and include `connectors/voice` in the domain filter.

## Tools

### 1. `voice_speak` — Text-to-Speech

**Type:** `@PlatformMutation` (produces audio output)
**Path:** `/api/connectors/voice/speak`

**Parameters:**
| Param | Type | Required | Default | Description |
|-------|------|----------|---------|-------------|
| `text` | String | yes | — | Text to synthesise |
| `voice` | String | no | default TTS | Voice ID, alias, or platform name. Resolved via `RoutingVoicePlatformService.resolve()` |
| `language` | String | no | null | Language hint (e.g. "en-GB") |
| `audioFormat` | String | no | "wav" | Output format (`@DefaultValue("wav")`) |
| `includePhonemes` | boolean | no | false | Include phoneme timing data (`@DefaultValue("false")`) |

**Resolution logic:**
1. If `voice` is provided → `routingService.resolve(voice)` (handles IDs, aliases, platform names)
2. If `voice` is null but `language` is set → `routingService.resolveQuery(VoiceQuery.builder().language(language).build())`
3. If both null → `routingService.defaultTts()`

**Return type:** `VoiceSynthesisResult` record:
```java
record VoiceSynthesisResult(
    String audioBase64,
    String audioFormat,
    String voiceId,
    String voiceName,
    String vendor,
    List<PhonemeTiming> phonemes
)
```

The `voiceId`/`voiceName`/`vendor` fields come from the resolved `VoiceDescriptor` — the agent sees which voice was actually used.

### 2. `voice_transcribe` — Speech-to-Text

**Type:** `@PlatformMutation` (accepts audio input)
**Path:** `/api/connectors/voice/transcribe`

**Parameters:**
| Param | Type | Required | Default | Description |
|-------|------|----------|---------|-------------|
| `audioBase64` | String | yes | — | Base64-encoded audio data |
| `audioFormat` | String | no | null | Audio format hint (e.g. "wav", "mp3") |
| `platform` | String | no | default STT | Platform ID to use |
| `languageHint` | String | no | null | Expected language |

**Resolution logic:**
1. If `platform` is provided → `platformService.platform(platform)`
2. If null → `routingService.defaultStt()`

**Return type:** `TranscriptionResult` directly (text, language, confidence) — already a clean DTO.

### 3. `voice_list_voices` — Voice Discovery

**Type:** `@PlatformQuery`
**Path:** `/api/connectors/voice/voices`

**Parameters:**
| Param | Type | Required | Default | Description |
|-------|------|----------|---------|-------------|
| `gender` | String | no | null | Filter by gender |
| `accent` | String | no | null | Filter by accent |
| `language` | String | no | null | Filter by language |
| `style` | String | no | null | Filter by style |
| `capability` | String | no | null | Filter by capability (TTS, STT, BOTH) |
| `vendor` | String | no | null | Filter by vendor |

**Return type:** `List<VoiceDescriptor>` — the existing model type serialises cleanly.

### 4. `voice_status` — Domain Report

Implemented via `DomainReportProvider`, not as a method on `ConnectorVoiceApi`. A separate `VoiceDomainReportProvider` bean produces a `DomainReport` with:
- Each registered `VoicePlatform` as a `ProviderReport` with its capabilities (STT, TTS, BOTH)
- Metadata: configured defaults (ttsVendor, sttVendor), voice count, alias count

The `DynamicToolRegistrar` auto-registers this as `connectors/voice_report`.

## DTOs

One new DTO in `connectors/graphql/dto/`:

**`VoiceSynthesisResult`** — wraps `SynthesisResult` with base64 audio and resolved voice metadata.

No DTO needed for transcription — `TranscriptionResult` from voice-spi is already clean.

## Error Handling

Use `McpCapabilityException.unsupported()` when a platform doesn't support the requested capability (STT or TTS). This produces a structured `McpOperationResult` that tells the agent which capabilities ARE available — enabling self-correction.

## Module Changes

**graphql/pom.xml:**
- Add compile dependency on `casehub-connectors-voice-spi`
- Add `connectors/voice` to the `-AdomainFilter` compiler arg

**ConnectorsModelEnricher:**
- Inject `VoicePlatformService`, add `voicePlatforms` count to `state()`

## Test Plan

- `ConnectorVoiceApiTest` — unit tests with mocked `RoutingVoicePlatformService` and `VoicePlatformService`
  - speak with default voice
  - speak with named voice/alias
  - speak with query params (language, gender)
  - speak includes phonemes when requested
  - transcribe with default platform
  - transcribe with named platform
  - list voices with filters
  - list voices all
  - unsupported capability returns structured McpOperationResult
- `VoiceDomainReportProviderTest` — unit test for report generation

## References

- connectors/graphql/ConnectorCalendarApi.java — pattern for @McpDomain API classes
- connectors/graphql/ConnectorHealthApi.java — pattern with capability checking
- connectors/voice-spi — SpeechToText, TextToSpeech, VoicePlatform, VoiceRegistry, RoutingVoicePlatformService
- platform/mcp-api — McpDomain, McpCapabilityException, DefaultValue, DomainReportProvider, McpEnrich
- platform/mcp/DynamicToolRegistrar.java — how domain operations become MCP tools
- MCP spec 2025-11-25 — AudioContent block (base64 + mimeType), tool result content types
