# HANDOFF — casehub-connectors

## Last Session

Landed 1 commit closing 1 issue. Published as `1ad4e87d` on main.

### What landed

| Issue | Fix | Module |
|-------|-----|--------|
| #246 | SSRF defense on `BotFrameworkClient.downloadAttachment` — host allowlist validation before authenticated GET, extracted `matchesHostPatterns` from `validateServiceUrl`, configurable `attachment-url-allowlist` | chat-teams |

### Code review findings filed

- #304 — Add request timeout and streaming size limit to attachment download (S/Low)
- #305 — Enforce HTTPS scheme in attachment download SSRF defense (XS/Low)

Blog entry revised: `blog/2026-11-18-mdp03-the-things-that-shouldnt-have-shipped.md` — added SSRF section with SVG diagram.

## Immediate Next Step

Branch `issue-303-textmark-marktiming-voice-spi` exists (empty, cut from main) for #303 (TextMark/MarkTiming in voice-spi — M/Med). This is a topic change from Teams hardening into voice pipeline mark timing.

Queue has 14 remaining items. Priority order: #303 (voice-spi marks, blocks agency#457), then #253 (teams-replyToMessage-user-prefix), then the rest.

## Context for Next Session

- The `matchesHostPatterns` method is now a shared `static` method in `BotFrameworkClient`. Both `validateServiceUrl` (service URL) and `downloadAttachment` (attachment URL) use it, but with separate allowlists. The 2-arg constructor delegates to the 3-arg, using the service URL allowlist for both — backward-compatible for existing tests.
- The attachment allowlist defaults to `*.sharepoint.com,graph.microsoft.com,*.botframework.com,smba.trafficmanager.net`. The service URL allowlist remains `*.botframework.com,smba.trafficmanager.net`.
- #304 tracks two pre-existing robustness gaps surfaced by the SSRF review: no request timeout and no max-attachment-bytes streaming limit (Discord equivalent has both).
- #305 tracks a gap in scheme enforcement: the host check passes `http://` URLs to allowed hosts, which would leak the Bearer token over plaintext.
- #303 design reference: agency `specs/issue-457-integrate-m2m-animation-library/decisions.md` D4 and D17.

## References

- `chat-teams/src/main/java/.../BotFrameworkClient.java` — SSRF defense at line 86, `matchesHostPatterns` at line 161
- `chat-teams/src/main/java/.../TeamsBeans.java` — `attachment-url-allowlist` config at lines 37–38 and 79–80
- `discord/src/main/java/.../DiscordClient.java` — reference SSRF implementation with `maxAttachmentBytes` streaming
