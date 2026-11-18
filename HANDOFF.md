# HANDOFF — casehub-connectors

## Last Session

Closed #320 (TTS marks), #322 (STT vocabularyHint). Transferred #321 to platform#885 (APT schema items gap — fix is in `yaml-plugin-processor`). Labelled all 38 open issues with scale/complexity. Scheduled 15 S/XS issues in queue order.

3 unpublished commits on canonical connectors main — publish when ready.

## In Progress

**#306 — LipSyncEnricher warmUp delegation** (XS/Low)
Branch: `feat/306-lipsync-warmup-delegation` — created, no changes yet.

**The fix:** `LipSyncEnricher` does not delegate `warmUp()` to its wrapped `TextToSpeech` delegate. `MarkTimingEnricher` already does this (line 33-34). One-line override needed:

```java
@Override
public void warmUp() {
    delegate.warmUp();
}
```

**Files:**
- Source: `voice-spi/src/main/java/io/casehub/connectors/voice/spi/LipSyncEnricher.java` (line 39, insert before closing brace)
- Test: `voice-spi/src/test/java/io/casehub/connectors/voice/spi/LipSyncEnricherTest.java` (add warmUp delegation test)
- Pattern to follow: `MarkTimingEnricher.java` lines 33-34

**Test approach:** Create a delegate with a `warmUp()` counter, wrap with `LipSyncEnricher`, call `warmUp()`, assert counter incremented. Same lambda-delegate pattern as the existing tests in `LipSyncEnricherTest`.

## Queue (15 items)

- [ ] #306 — lipsync-warmup-delegation (XS/Low) ← active
- [ ] #242 — teams-guard-driveId (XS/Low)
- [ ] #247 — teams-attachment-size-limit (XS/Low)
- [ ] #248 — teams-personal-rich-content (S/Med)
- [ ] #244 — teams-test-safeAttachmentName (XS/Low)
- [ ] #245 — teams-test-reply-attachments (XS/Low)
- [ ] #252 — teams-integration-test-proactive (S/Med)
- [ ] #255 — teams-invoke-handler-integration-test (S/Med)
- [ ] #267 — teams-settings-url-manifest (XS/Low)
- [ ] #266 — teams-comment-field-stripping (XS/Low)
- [ ] #268 — teams-cdi-wiring-guidance (XS/Low)
- [ ] #265 — docs-messaging-extensions-xref (XS/Low)
- [ ] #254 — docs-webhook-responded-invoke-handler (S/Low)
- [ ] #241 — docs-user-scoped-accessor-guides (S/Low)
- [ ] #218 — sharepoint-multi-site (S/Low)

## References

- Blog: `blog/2026-11-18-mdp04-the-word-the-gesture-needs.md`
- LipSyncEnricher: `voice-spi/src/main/java/io/casehub/connectors/voice/spi/LipSyncEnricher.java`
- MarkTimingEnricher (warmUp pattern): `voice-spi/src/main/java/io/casehub/connectors/voice/spi/MarkTimingEnricher.java`
- Platform issue: casehubio/platform#885 (APT schema items)
