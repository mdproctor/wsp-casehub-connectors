# HANDOFF — casehub-connectors

## Last Session

Closed #320 — added `@Optional List<TextMark> marks` to `TtsSynthesisePlugin` so YAML pipeline consumers can request mark-timed synthesis. Landed as `d6e9b99a`. Revised the mark-timing blog (`2026-11-18-mdp04`) with a "last mile" section explaining the pipeline gap and a flow diagram SVG. Filed #321 (APT schema items gap — generated JSON schema for `List<T>` plugin fields lacks `items` definition) and #322 (audit `SttTranscribePlugin` for similar missing optional fields). Both XS/Low.

Canonical connectors repo has 1 unpublished commit on main — publish when ready.

## Immediate Next Step

#321 — APT-generated JSON schema for plugin `List` fields lacks items definition. XS/Low.

## Queue

- [ ] #321 — apt-schema-list-items (XS/Low)
- [ ] #322 — stt-plugin-optional-fields-audit (XS/Low)

## References

- Blog: `blog/2026-11-18-mdp04-the-word-the-gesture-needs.md`
- Blog SVG: `blog/images/2026-11-18-pipeline-mark-flow.svg`
- Test: `integration-tests/src/test/java/io/casehub/connectors/it/VoiceRoundTripTest.java`
- Plugin: `voice-pipeline-steps/src/main/java/io/casehub/connectors/voice/steps/TtsSynthesisePlugin.java`
