# Voice Provider VoicePlatform Implementations

**Issue:** casehubio/connectors#202
**Date:** 2026-11-10
**Scope:** KokoroVoicePlatform (TTS) and WhisperVoicePlatform (STT) in voice-sherpa

## Context

The voice manifest and routing infrastructure is complete: `VoiceManifest`, `VoiceManifestLoader`, `RoutingVoicePlatformService`, `VoiceConfigProducer`, pipeline steps. What's missing is `VoicePlatform` implementations that bridge the sherpa engines to the routing system.

The manifest already supports separate `tts-vendor` and `stt-vendor` defaults, so each engine should be its own `VoicePlatform` — not bundled. This enables mix-and-match: a deployment can use Kokoro for TTS and Whisper for STT, or swap either independently.

## Design

### KokoroVoicePlatform

Plain Java class in `voice-sherpa`. Implements `VoicePlatform` and `AutoCloseable`.

- **ID:** `"kokoro"`
- **Capability:** TTS only. `supports(TextToSpeech.class)` → true, `supports(SpeechToText.class)` → false.
- **Construction:** Takes `KokoroConfig` (model dir, voice ID, threads). Directly constructs and owns a `KokoroTextToSpeech` instance.
- **Factory:** `withDefaults()` auto-provisions the native lib and kokoro model via `Provisioner`.
- **Lifecycle:** `close()` delegates to `KokoroTextToSpeech.close()`, releasing native sherpa-onnx resources.
- **speechToText():** Throws `UnsupportedCapabilityException` — this is a TTS-only platform.
- **textToSpeech():** Returns the owned `KokoroTextToSpeech`.

### WhisperVoicePlatform

Plain Java class in `voice-sherpa`. Implements `VoicePlatform` and `AutoCloseable`.

- **ID:** `"whisper"`
- **Capability:** STT only. `supports(SpeechToText.class)` → true, `supports(TextToSpeech.class)` → false.
- **Construction:** Takes `SherpaConfig` (model dir, threads). Directly constructs and owns a `SherpaOnnxSpeechToText` instance.
- **Factory:** `withDefaults()` auto-provisions the native lib and whisper-tiny model via `Provisioner`.
- **Lifecycle:** `close()` delegates to `SherpaOnnxSpeechToText.close()`, releasing native sherpa-onnx resources.
- **speechToText():** Returns the owned `SherpaOnnxSpeechToText`.
- **textToSpeech():** Throws `UnsupportedCapabilityException` — this is an STT-only platform.

### Seed Catalog Update

Update `voice-spi/src/main/resources/voices/seed-catalog.yaml` to declare both providers and set Kokoro as default TTS, Whisper as default STT:

```yaml
providers:
  - vendor: ref
  - vendor: kokoro
  - vendor: whisper

voices:
  af_heart:
    vendor: kokoro
    voice-id: "0"
    name: Heart
    gender: female
    accent: american
    quality: 4.5
    languages: [en-US]
    capability: tts
  bm_daniel:
    vendor: kokoro
    voice-id: "1"
    name: Daniel
    gender: male
    accent: british
    quality: 4.3
    languages: [en-GB]
    capability: tts

defaults:
  tts-vendor: kokoro
  stt-vendor: whisper
```

Voices now reference vendor `kokoro` instead of `ref`. The `voice-id` maps to Kokoro's integer voice IDs.

### What's Not In Scope

- **CDI integration** — voice-sherpa stays pure Java (no Jakarta CDI). Consumers wire platforms via producers or ServiceLoader. Follow-up issue.
- **Streaming STT** — `VoicePlatform.speechToText()` returns `SpeechToText` (batch). Streaming (`WhisperSpeechToText` via whisper.cpp) is accessed directly, not through the platform SPI.
- **Cloud providers** (OpenAI TTS, Google Cloud TTS) — separate VoicePlatform implementations, separate issues.
- **Voice registration playbook step** — follow-up for CosyVoice3 voice cloning through playbooks.
- **Credential resolution** — `VoiceProviderDeclaration.credential` field exists but wiring to `CredentialResolver` is a follow-up when cloud providers are added.

## Testing

Plain JUnit 5 + AssertJ (no Quarkus test runtime — voice-sherpa is pure Java).

- `KokoroVoicePlatformTest`: id, supports (TTS yes, STT no), speechToText throws, textToSpeech returns non-null, close delegates
- `WhisperVoicePlatformTest`: id, supports (STT yes, TTS no), textToSpeech throws, speechToText returns non-null, close delegates
- Seed catalog loading: existing `VoiceManifestLoaderTest` should verify the updated catalog parses correctly with kokoro/whisper providers

Integration tests (native lib required) are guarded by `@EnabledIf` checks on `SherpaLibrary.isAvailable()` / `WhisperLibrary.isAvailable()` — consistent with existing voice-sherpa test patterns.

## References

- `voice-spi/…/VoicePlatform.java` — the SPI being implemented
- `voice-spi/…/RoutingVoicePlatformService.java` — routing layer that discovers platforms
- `voice-spi/…/VoiceManifestDefaults.java` — separate tts-vendor / stt-vendor
- `voice-ref/…/ReferenceVoicePlatform.java` — reference implementation pattern
- `voice-sherpa/…/KokoroTextToSpeech.java` — Kokoro engine
- `voice-sherpa/…/SherpaOnnxSpeechToText.java` — Whisper STT via sherpa-onnx
- `voice-sherpa/pom.xml` — pure Java, no CDI
