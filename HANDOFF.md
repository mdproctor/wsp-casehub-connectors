# HANDOFF — casehub-connectors

## Last Session

Closed #319 — added three mark-timing integration tests to `VoiceRoundTripTest`: round-trip with bounds and ordering, `MarkTimingEnricher` pass-through with the real ref platform, and out-of-bounds `charOffset` clamping. Landed as `1fb1c3c9`. Revised the mark-timing blog (`2026-11-18-mdp04`) to replace the aspirational testing section with actual coverage details and a paragraph on why integration tests catch decorator composition bugs that unit tests with mocks cannot. Filed #320 — `TtsSynthesisePlugin` constructs `SynthesisOptions` without marks, so pipeline consumers can't request mark-timed synthesis.

## Immediate Next Step

#320 — add an optional `marks` input to `TtsSynthesisePlugin` and pass it through to `SynthesisOptions`. XS/Low.

## References

- Blog: `blog/2026-11-18-mdp04-the-word-the-gesture-needs.md`
- Test: `integration-tests/src/test/java/io/casehub/connectors/it/VoiceRoundTripTest.java`
- Plugin: `voice-pipeline-steps/src/main/java/io/casehub/connectors/voice/steps/TtsSynthesisePlugin.java`
