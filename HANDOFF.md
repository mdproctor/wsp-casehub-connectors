# HANDOFF — casehub-connectors

## Last Session

Implementation session for #188 (real-time inbound channels). All 7 plan tasks implemented: voice-spi, voice-ref, DeferredDelivery WebhookResult, WhatsApp media download, email-google-inbound (Gmail push), voice-pipeline-steps (stt-transcribe), and SherpaVoicePlatform adapter in agency. 4-fork code review caught 8 issues (SSRF, Base64 encoding, history pagination, watch lifecycle, type fallback) — all fixed. Integration test module added: 21 tests covering voice TTS→STT round-trip, YAML-loaded pipeline via Walker/PluginScanner/StreamingStepFactory, WhatsApp media flow, Gmail push flow. Blog entry written: "When Your Webhook Needs Two Speeds." Filed #192 (JWT JWKS verification), #193 (phoneme suppression). Closed #194 (streaming plugin compat — proven working).

**CRITICAL OUTSTANDING WORK:** The SherpaVoicePlatform in agency was implemented as an adapter bridge with parallel type hierarchies — connectors has `SpeechToText`/`TextToSpeech` + model types, agency has its own `SpeechToTextService`/`TextToSpeechService` + model types, and the adapter maps field-by-field between them. This is wrong. Agency should use the connectors voice types directly. The adapter must be replaced by a refactor-move.

## Immediate Next Step

**Migrate agency speech types to connectors voice-spi.** This is a refactor-move, not an adapter. Use `ide_open_workspace` with both repos, then `ide_refactor_rename` and `ide_move_file` for the migration.

The migration:
1. Enrich connectors `voice-spi` types to cover agency's needs: add `transcribe(Path, TranscriptionOptions)` overload to `SpeechToText`, add `phonemes` field to `SynthesisResult`
2. Delete agency's 6 duplicate model types from `speech-api` (`TranscriptionOptions`, `TranscriptionResult`, `SynthesisOptions`, `SynthesisResult`, `SpeechToTextService`, `TextToSpeechService`)
3. Make all agency implementations (3 STT classes: `SherpaOnnxSpeechToText`, `SherpaOnnxStreamingSpeechToText`, `WhisperSpeechToText`; 5 TTS classes: `SherpaOnnxTextToSpeech`, `VitsTextToSpeech`, `KokoroTextToSpeech`, `Audio8TextToSpeech`, `TtsPipeline`) directly implement connectors `SpeechToText`/`TextToSpeech`
4. Update all imports across 6 agency modules (~90 source files, ~50 test files): `speech-api`, `speech-sherpa`, `speech-ws`, `speech-demo`, `agentic-yaml`, `agency-core`
5. Delete the `SherpaVoicePlatform` adapter (no longer needed — sherpa classes implement VoicePlatform capabilities directly)
6. Build and test both repos

**After migration:** Create `tts-synthesise` `@Plugin(streaming=true)` step type, then write a YAML pipeline integration test that chains `tts-synthesise → stt-transcribe` with `FusionPass` fusion — proving real TTS→STT data flow through YAML-loaded pipeline with no manual wiring.

Also: #192 (JWT JWKS verification for Gmail push) and #193 (suppress includePhonemes in bridge — moot after migration).

## Cross-Module

- agency commit `b743520b` on main adds `SherpaVoicePlatform` adapter + `casehub-connectors-voice-spi` dependency to `speech-sherpa/pom.xml` — this commit needs to be replaced by the migration
- Blast radius: 6 agency modules, ~90 source files, ~50 test files
- Agency's speech-api has ~25 types total, but only 6 overlap with connectors voice-spi (the core STT/TTS interfaces + 4 model records). The remaining ~19 types (streaming STT, denoising, VAD, speaker ID, diarization, phoneme alignment, avatar cognition) stay in agency

## References

- `specs/issue-188-real-time-inbound-channels/2026-10-30-real-time-inbound-channels-design.md` — approved design spec
- `plans/2026-10-30-real-time-inbound-channels.md` — implementation plan (4 batches, 7 tasks — all done)
- `blog/2026-10-31-mdp01-when-your-webhook-needs-two-speeds.md` — blog entry
- `integration-tests/` — 21 integration tests (voice round-trip, WhatsApp flow, Gmail push, YAML pipeline)
- Agency speech-api types to migrate: `/Users/mdproctor/claude/casehub/agency/speech-api/src/main/java/io/casehub/agency/speech/`
- Agency speech-sherpa implementations: `/Users/mdproctor/claude/casehub/agency/speech-sherpa/src/main/java/io/casehub/agency/speech/sherpa/`
- GitHub issues: #192 (JWT JWKS), #193 (phoneme suppression)
