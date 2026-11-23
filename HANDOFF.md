# Handoff — connectors

## Session summary

- **#345 — LocalDate step coercion**: Deferred to platform repo. Filed casehubio/platform#938 with root cause analysis and fix options. No connectors code changes needed.
- **#346 — Voice-ref speaker discrimination**: Fixed. Replaced `hashCode()` nibble extraction (8-dim [0,1]) with SHA-256 (16-dim [-1,1]) in `RefSpeakerEmbeddingExtractor`. Added 4 unit tests verifying embedding distinctness and registry discrimination including TTS WAV round-trip.

## Cross-repo state

| Repo | Branch | Status |
|------|--------|--------|
| connectors | issue-346-voice-ref-speaker-discrimination | 1 commit ahead of main |
| wsp-connectors | issue-346-voice-ref-speaker-discrimination | queue updates, handoff |

## Queue

Both items done. Queue is empty after landing.

## Notes

- Playbook joint-account tests still assert `identified: true` only (not speaker name) because `voice-sherpa`'s `yaml-services.yaml` overrides ref implementations on the test classpath — the Campplus extractor can't discriminate the ref TTS's fake audio. The ref-level unit tests verify name discrimination directly.
- platform#938 needs to be picked up in a platform session for the LocalDate fix
