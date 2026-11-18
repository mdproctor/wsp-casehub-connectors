# HANDOFF — casehub-connectors

## Last Session

Landed 3 commits closing 2 issues on branch `issue-317-clamp-charOffset-mark-timing`. Published to both remotes.

### What landed

| Issue | Change | Module |
|-------|--------|--------|
| #317 | Clamp `mark.charOffset()` to `text.length()` in `ReferenceVoicePlatform` and both `MarkTimingEnricher` paths (phoneme-proportional and character-proportional) — standardises with VITS which already clamped | voice-ref, voice-spi |
| #318 | Replace "Proactive messaging" with "User channel operation" in `resolveConversation` error messages — method is called from both send and reply paths | chat-teams |

Blog entry: revised `docs/blog/2026-11-18-mdp04-the-word-the-gesture-needs.md` — added "The contract every tier must keep" section on input validation consistency across decorator fallback tiers, with new SVG diagram.

### Follow-up issue filed

| # | Title | Scale | Complexity |
|---|-------|-------|------------|
| #319 | Add mark-timing round-trip coverage to `VoiceRoundTripTest` — clamp behavior tested at unit level but no integration coverage | XS | Low |

### Code review findings

- Test assertions tightened from `isLessThanOrEqualTo` to `isEqualTo` — squashed into the primary fix
- Audit confirmed: all 4 `charOffset` usage sites now clamp (VITS, Ref, Enricher phoneme, Enricher char)
- Teams error messages in `sendPersonalMessage`/`replyToMessage` correctly differentiate callers — not the same bug as #318

## Immediate Next Step

Queue has 1 remaining item:

| # | Title | Scale | Complexity |
|---|-------|-------|------------|
| #319 | mark-timing-round-trip-integration-test | XS | Low |

## Context for Next Session

- The VITS guarded-exact path (#307) is the Tier 1 implementation from the D4 animation design spec. It unblocks casehub-agency#457 (gesture timing). The `MarkTimingEnricher` provides Tier 2/3 fallback for engines that return no marks.
- All `charOffset` usage sites now clamp consistently. The `TextMark` record validates `charOffset >= 0` but intentionally has no upper bound — the clamp-at-usage-site pattern is the design choice.
- `VoiceRoundTripTest` has zero mark-timing coverage. #319 should add: round-trip with marks, enricher pass-through, and out-of-bounds charOffset at integration level.
