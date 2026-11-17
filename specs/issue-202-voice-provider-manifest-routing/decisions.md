# Decisions — Voice Provider Manifest and Routing (#202)

## D1: Per-engine VoicePlatform, not bundled wrapper

**Choice:** Each engine (Kokoro, Whisper, CosyVoice3, etc.) gets its own `VoicePlatform` implementation. The routing layer mixes and matches via `tts-vendor`/`stt-vendor` defaults.
**Alternatives:**
- Bundled `SherpaVoicePlatform` wrapping both STT+TTS — artificial coupling, prevents mix-and-match
- Eliminate `VoicePlatform`, route to `SpeechToText`/`TextToSpeech` directly — breaks pattern every other platform SPI follows
**Rationale:** A vendor may have great TTS but no STT, or vice versa. Mix-and-match is the natural model. The routing layer (`VoiceManifestDefaults.ttsVendor`/`sttVendor`, `RoutingVoicePlatformService`) already supports this.
**Trade-offs:** Each platform is a thin class (one capability, one engine). More classes, but each is simple and self-contained.
**Sources:** `voice-spi/…/VoicePlatform.java`, `voice-spi/…/RoutingVoicePlatformService.java`, `voice-spi/…/VoiceManifestDefaults.java`, `voice-ref/…/ReferenceVoicePlatform.java`
**Exploration:** deep-analysis (conversation)
**Status:** captured

## D2: Composition root, not wrapper

**Choice:** Each VoicePlatform directly constructs and owns its engine from config. Not wrapping externally-created instances.
**Alternatives:**
- Adapter/wrapper around pre-built instances — technical debt, extra layer
- Have existing classes (KokoroTextToSpeech, SherpaOnnxSpeechToText) directly implement VoicePlatform — can't, they're single-capability
**Rationale:** The platform IS the thing, not a pass-through. It reads config, provisions models, creates the engine, manages lifecycle.
**Trade-offs:** None significant — this is the standard provider pattern.
**Sources:** `voice-ref/…/ReferenceVoicePlatform.java`, `bank-truelayer` pattern
**Exploration:** quick
**Depends on:** D1
**Status:** captured

## D3: Kokoro as default TTS, Heart voice

**Choice:** `KokoroVoicePlatform` (id: `"kokoro"`) is the default TTS provider. Default voice: `af_heart` (Heart).
**Alternatives:**
- CosyVoice3 via TtsPipeline — higher quality voice cloning but heavier
- Vits/Piper — lighter but lower quality
**Rationale:** Kokoro is the project default. Heart is already in the seed catalog.
**Trade-offs:** No voice cloning (that's CosyVoice3's strength). Kokoro has 28 built-in voices.
**Sources:** `voice-sherpa/…/KokoroTextToSpeech.java`, `voice-spi/src/main/resources/voices/seed-catalog.yaml`
**Exploration:** quick
**Status:** captured

## D4: Whisper (via sherpa-onnx) as default STT

**Choice:** `WhisperVoicePlatform` (id: `"whisper"`) is the default STT provider. Uses `SherpaOnnxSpeechToText` (batch `SpeechToText` interface via sherpa-onnx).
**Alternatives:**
- `WhisperSpeechToText` via whisper.cpp — streaming but implements `StreamingSpeechToText`, not `SpeechToText`
**Rationale:** `VoicePlatform.speechToText()` returns `SpeechToText` (batch). `SherpaOnnxSpeechToText` fits. Streaming STT is a separate concern, accessed directly.
**Trade-offs:** No streaming through the platform SPI. Consumers needing streaming access `WhisperSpeechToText` directly.
**Sources:** `voice-sherpa/…/SherpaOnnxSpeechToText.java`, `voice-spi/…/SpeechToText.java`, `voice-spi/…/StreamingSpeechToText.java`
**Exploration:** quick
**Depends on:** D1
**Status:** captured

## D5: Plain Java, no CDI in voice-sherpa

**Choice:** Platform classes are plain Java in voice-sherpa (Java 22). CDI integration is a separate concern — not part of this issue.
**Alternatives:**
- Add CDI annotations to voice-sherpa — pollutes pure native module
- Separate CDI module — follow-up work
**Rationale:** voice-sherpa is intentionally CDI-free (FFM/Panama). CDI wiring can be added later via a producer or ServiceLoader.
**Trade-offs:** Platforms aren't auto-discovered by Quarkus CDI. Consumer apps need explicit wiring. This is a follow-up issue.
**Sources:** `voice-sherpa/pom.xml` (no Jakarta EE deps), `voice-spi/…/VoiceConfigProducer.java`
**Exploration:** quick
**Status:** captured
