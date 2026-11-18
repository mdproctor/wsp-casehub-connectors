# HANDOFF — casehub-connectors

## Last Session

Landed 6 commits closing 7 issues across 4 modules. Published as `833a8796` on main.

### What landed

| Issue | Fix | Module |
|-------|-----|--------|
| #197 | Already resolved — TTS text encoding was implemented in earlier voice platform work | voice-ref |
| #240 | Reuse `NetHttpTransport` as constructor-initialised field (matches Calendar/Contacts pattern) | document-google, email-google |
| #192 | Full JWT signature verification via jose4j — `JwtValidator` SPI + `GoogleOidcJwtValidator` with JWKS caching | email-google-inbound |
| #251 | Guard personal channel refs — graceful degradation for listMembers, reactions, messageHistory, memberManagement | chat-teams |
| #249 | Return real activity ID from `sendToConversation`/`replyToActivity` (void→String) | chat-teams |
| #250 | Fix nested HTML tags in `htmlToMarkdown` — `[^<]*` → `.*?` | chat-teams |
| #243 | Guard `uploadToChannelFolder` against zero-length content | chat-teams |

### Code review findings filed

- #299 — `ChannelManagement.delete/find` not guarded for personal channels (XS/Low)
- #300 — BotFramework error propagation changed silently — needs retry/error strategy (S/Med)
- #301 — `htmlToMarkdown` same-tag nesting edge case (XS/Low)
- #302 — `JwtValidator` CDI qualifier to prevent future ambiguity (XS/Low)

Blog entry: `blog/2026-11-18-mdp03-the-things-that-shouldnt-have-shipped.md`

## Immediate Next Step

Queue has 12 remaining items. Next unchecked: #246 (teams-ssrf-attachment-download) — security fix for SSRF in Teams attachment downloading. Then #253, #293, and the newly filed review findings (#299–#302).

## Context for Next Session

- The `BotFrameworkClient.postJson` (fire-and-forget) method was removed — all Bot Framework calls now go through `postJsonWithResponse` which throws on non-2xx. Issue #300 tracks whether retry logic is needed for transient errors.
- `JwtValidator` is a bare CDI interface with no qualifier. Currently safe (single producer in `GmailPushBeans`), but #302 tracks adding a qualifier before a second producer appears.
- The Teams personal channel guard pattern is consistent across messaging, threading, members, reactions, memberManagement, and messageHistory. `ChannelManagement.delete/find` are the remaining unguarded operations (#299).

## References

- `email-google-inbound/src/main/java/.../GoogleOidcJwtValidator.java` — jose4j-based JWKS verifier
- `chat-teams/src/main/java/.../BotFrameworkClient.java` — now uses `postJsonWithResponse` for all send/reply
- `chat-teams/src/main/java/.../TeamsChatPlatform.java` — personal channel guards at lines 158, 166, 382, 392, 400, 409, 417
