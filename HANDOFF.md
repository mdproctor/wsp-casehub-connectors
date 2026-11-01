# HANDOFF — casehub-connectors

## Last Session

Voice type migration (#188) — completed full migration of all voice types from agency to connectors voice-spi. Two batches:

**Batch 1 (7 types):** Core STT/TTS interfaces + model records. Enriched connectors voice-spi with PhonemeTiming, phonemes on SynthesisResult, transcribe(Path) default method, convenience methods. Deleted agency duplicates (SpeechToTextService, TextToSpeechService, TranscriptionResult, TranscriptionOptions, SynthesisResult, SynthesisOptions, PhonemeTiming). Deleted SherpaVoicePlatform adapter. Created tts-synthesise @Plugin(streaming=true) + YAML chain integration test.

**Batch 2 (14 types):** Streaming STT (StreamingSpeechToText — renamed from StreamingSpeechToTextService, RecognitionStream), phoneme/lip-sync (PhonemeAligner, LipSyncEnricher), audio processing (SpeechDenoiser, StreamingSpeechDenoiser, StreamingSpeechDenoiserFactory, VoiceActivityFilter, VoiceActivityFilterFactory), speaker ID (SpeakerEmbedding, SpeakerMatch, SpeakerEmbeddingExtractor, SpeakerRegistry, VoiceprintStore).

**Deep audit:** 3 parallel forks verified zero remnant references in Java code, all 21 types exist with correct signatures, all 16 agency implementations wired to connectors interfaces. Fixed: 4 SPI contract tests moved to connectors (31 voice-spi tests now), 13 stale doc references updated, file/class name mismatch fixed.

**Agency landed:** Squashed to single commit `810dbe37` on agency main. Branch stamped closed. #193 closed (moot — adapter deleted).

## Immediate Next Step

**Voice playbook integration tests** — end-to-end YAML playbook examples showing voice capabilities with simulated services. Design agreed:

- `examples/voice-interaction-patterns/` directory with numbered YAMLs (platform pattern)
- Progressive: 01-voice-echo (STT→TTS), 02-voice-agent (STT→agent→TTS), 03-banking-assistant (STT→agent→BankPlatform→TTS), 04-travel-booking, 05-health-appointment
- `VoiceExampleExecutionTest` modeled on platform's `AgentExampleExecutionTest`
- Execution harness bridges PluginStep (voice) + InvokeStep (agent) — neither `executeAll()` nor `StructuralStepEvaluator` handles both alone
- CorpusBackend for simulated LLM with scenario-specific canned responses
- ReferenceVoicePlatform + bank-ref/travel-ref/health-ref for simulated platforms
- Need to add deps: bank-ref, travel-ref, health-ref, calendar-ref, email-ref, location-ref, playbook-compiler, mutiny (all test scope)

**Future:** YAML-based assertion mechanism for self-verifying playbooks (assert: steps in YAML).

## Cross-Module

- Agency commit `810dbe37` on main — voice type migration landed
- Agency speech-api retains 12 types: text cleanup (TextFilter, CleanupConfig, CorrectionStrategy), avatar/conversation (6), diarization (3)
- #192 (JWT JWKS verification for Gmail push) remains open

## References

- Connectors branch: `issue-188-real-time-inbound-channels` (paused)
- `specs/issue-188-real-time-inbound-channels/2026-10-30-real-time-inbound-channels-design.md`
- `plans/2026-10-30-real-time-inbound-channels.md`
- `integration-tests/` — 22 integration tests, 31 voice-spi tests
- Platform pattern: `platform/examples/agent-interaction-patterns/` + `AgentExampleExecutionTest`
- Key infrastructure: `StructuralStepEvaluator`, `CorpusBackend`, `ExampleEnvironment`, `InvokeHandler`
