# HANDOFF — casehub-connectors

## Last Session

Landed 1 commit closing 1 issue. Landed as `aee48155` on main (unpublished).

### What landed

| Issue | Change | Module |
|-------|--------|--------|
| #303 | SSML-style mark timing for speech-gesture synchronisation — `TextMark`/`MarkTiming` model types, `SynthesisOptions.marks`/`SynthesisResult.marks` fields with backward-compatible constructors, `MarkTimingEnricher` decorator with phoneme-proportional and character-proportional fallback tiers, `warmUp()` delegation | voice-spi |

### Follow-up issues filed

| # | Title | Scale | Complexity |
|---|-------|-------|------------|
| #306 | LipSyncEnricher warmUp delegation | XS | Low |
| #307 | VITS guarded-exact mark resolution in VitsTextToSpeech | M | Med |
| #308 | voice-ref mark timing support for round-trip testing | XS | Low |

Blog entry written: `blog/2026-11-18-mdp04-the-word-the-gesture-needs.md` — mark timing design, three-tier fallback strategy, ownership split between server speech time and client animation time.

### Code review findings addressed

- warmUp() not delegated in MarkTimingEnricher — fixed in-session
- Mixed static/instance on fallback methods — fixed (characterProportional made static)
- Missing boundary edge-case tests — added 5 tests (single-char text, mark at text end, empty text)
- LipSyncEnricher has the same warmUp delegation gap — filed as #306

## Immediate Next Step

Queue has 16 remaining items starting with #253 (teams-replyToMessage-user-prefix). The three follow-ups (#306, #307, #308) are appended at the end.

#307 (VITS guarded-exact) blocks casehub-agency#457 (gesture timing D4). Design reference: agency `specs/issue-457-integrate-m2m-animation-library/decisions.md` D4 and D17. This is the highest-value follow-up but requires espeak integration in `voice-sherpa`, not just voice-spi.

## Context for Next Session

- `SynthesisOptions` and `SynthesisResult` are both records with new fields added as the last component. Backward-compatible constructors (4-arg for Options, 2-arg and 3-arg for Result) mean all existing callers compile unchanged — verified across the full project.
- `MarkTimingEnricher` follows the same decorator pattern as `LipSyncEnricher` — wraps a `TextToSpeech` delegate. The two enrichers compose: LipSync can wrap the engine for phonemes, then MarkTiming can wrap the result for mark timings from those phonemes.
- The enricher's phoneme-proportional path snaps mark offsets to the nearest phoneme start boundary (forward snap). The character-proportional path is a simple linear proportion over WAV duration. Both return `List.of()` for empty text or invalid WAV data.
- 1 unpublished commit on connectors main — needs `canonical_sync.py publish` or manual push.

## References

- `voice-spi/src/main/java/.../model/TextMark.java` — caller-specified character positions for mark timing
- `voice-spi/src/main/java/.../model/MarkTiming.java` — engine-returned word onset times
- `voice-spi/src/main/java/.../spi/MarkTimingEnricher.java` — three-tier fallback decorator
- `voice-spi/src/main/java/.../spi/LipSyncEnricher.java` — updated to preserve marks through enrichment
