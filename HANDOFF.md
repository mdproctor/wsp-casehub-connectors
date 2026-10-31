# HANDOFF — casehub-connectors

## Last Session

Completed voice type migration (connectors#188). All 3 phases executed:

**Phase 1 — Enriched connectors voice-spi:** Added `PhonemeTiming` record, `phonemes` field on `SynthesisResult` (with backward-compatible 2-arg ctor), `transcribe(Path)` default method on `SpeechToText`, null check on `TranscriptionResult`, `withVocabularyHint()` on `TranscriptionOptions`, `defaults()` on `SynthesisOptions`. All existing tests pass unchanged.

**Phase 2 — Migrated agency to connectors voice-spi (53 files, 6 modules):** Deleted 7 duplicate types from agency `speech-api` (SpeechToTextService, TextToSpeechService, TranscriptionResult, TranscriptionOptions, SynthesisResult, SynthesisOptions, PhonemeTiming). All implementations now directly implement connectors interfaces: `SherpaOnnxSpeechToText implements SpeechToText`, all TTS classes `implement TextToSpeech`. Added `transcribe(byte[], TranscriptionOptions)` method to `SherpaOnnxSpeechToText` (delegates via temp file). Deleted `SherpaVoicePlatform` adapter and test — no longer needed. Agency builds green: 1897 tests pass.

**Phase 3 — Created tts-synthesise plugin + YAML chain test:** `TtsSynthesisePlugin` `@Plugin(streaming=true)` synthesises text to audio via VoicePlatform TTS. YAML pipeline integration test chains `tts-synthesise → stt-transcribe`, resolves both from YAML via Walker/PluginScanner, runs FusionPass, then proves real TTS→STT data flow. 22 integration tests pass.

**Agency branch:** `issue-188-voice-type-migration` at commit `39823b8a` — needs PR to casehubio/agency.

## Immediate Next Step

Agency migration branch needs PR and merge. Then close #193 (phoneme suppression — moot after migration since connectors `SynthesisResult` has phonemes natively). #192 (JWT JWKS verification for Gmail push) remains open.

## References

- Connectors branch: `issue-188-real-time-inbound-channels`
- Agency branch: `issue-188-voice-type-migration`
- `specs/issue-188-real-time-inbound-channels/2026-10-30-real-time-inbound-channels-design.md`
- `plans/2026-10-30-real-time-inbound-channels.md`
- `blog/2026-10-31-mdp01-when-your-webhook-needs-two-speeds.md`
- `integration-tests/` — 22 integration tests
- GitHub issues: #192 (JWT JWKS), #193 (phoneme suppression — moot)
