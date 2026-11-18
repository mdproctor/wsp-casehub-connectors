# HANDOFF — casehub-connectors

## Last Session

Landed 6 commits closing 5 issues on branch `issue-308-voice-ref-mark-timing`. Published to both remotes.

### What landed

| Issue | Change | Module |
|-------|--------|--------|
| #308 | Native mark timing in `ReferenceVoicePlatform` — 50ms-per-character synthetic duration for round-trip testing without native deps | voice-ref |
| #307 | VITS guarded-exact mark timing in `VitsTextToSpeech` — espeak phonemisation, space-token word boundaries, consistency guard (`prefixWords + suffixWords == spaceTokens + 1`), monotonicity check | voice-sherpa |
| #312 | `isUserChannel` guards on all 9 Teams chat operations (listMembers, listMessageHistory, reactions add/remove/list, memberManagement add/remove, channelManagement delete/find) | chat-teams |
| #313 | Extract `resolveConversation(aadObjectId)` from duplicated logic in `sendProactiveMessage`/`replyToUserChannel` | chat-teams |
| #314 | Include URL in `BotFrameworkClient.checkResponse` exception body for diagnostics | chat-teams |

### Follow-up issues filed

| # | Title | Scale | Complexity |
|---|-------|-------|------------|
| #317 | Clamp `charOffset` in mark timing to `text.length()` — ref impl doesn't clamp, VITS does, inconsistent | XS | Low |
| #318 | `resolveConversation` error messages say "Proactive messaging" for both send and reply paths | XS | Low |

Blog entry: revised `blog/2026-11-18-mdp04-the-word-the-gesture-needs.md` — added VITS guarded-exact algorithm explanation, consistency guard rationale, new SVG diagram, ref testing section.

### Code review findings addressed

- `TeamsChannelManagement.delete` and `find` missing `isUserChannel` guard — fixed (same gap as #312, missed from original scope)
- Ref impl `charOffset` clamping inconsistency with VITS path — filed as #317
- `resolveConversation` error message wording — filed as #318

## Immediate Next Step

Queue has 2 remaining items:

| # | Title | Scale | Complexity |
|---|-------|-------|------------|
| #317 | clamp-charOffset-mark-timing | XS | Low |
| #318 | resolveConversation-error-messages | XS | Low |

Both are XS review follow-ups from this session.

## Context for Next Session

- The VITS guarded-exact path (#307) is the Tier 1 implementation from the D4 animation design spec. It unblocks casehub-agency#457 (gesture timing). The `MarkTimingEnricher` provides Tier 2/3 fallback for engines that return no marks.
- `extractPhonemeTimings` was refactored to share dur frame extraction with the new mark timing path — same behavior, one fewer tensor read.
- The ref implementation (#308) uses a synthetic 50ms-per-character duration rather than parsing the WAV header (the ref WAV payload is text, not audio samples, so the computed duration would be near-zero).
- `resolveConversation` (#313) unified the store-lookup → cold-start → store-back pipeline. Error messages currently say "Proactive messaging" for both the send and reply paths — cosmetic, filed as #318.
