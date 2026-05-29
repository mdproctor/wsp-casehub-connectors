# Handoff — connectors

## Last Session

Completed issues #7 (email inbound) and #8 (v1 polish). New `email-inbound`
Maven module: `EmailInboundConnector` (IMAP polling, per-account daemon executors,
at-least-once SEEN), `EmailInboundAccountProvider` SPI (@DefaultBean reads
@ConfigProperty), `ContentExtractor` (recursive MIME). 108 tests total, 24 new.
Issue #8: structured XFF-sanitized security log, `extractChallenge` refactor in
Slack (eliminates double JSON parse), metadata populated for Slack/Twilio/WhatsApp.
Both issues closed, squashed (11→3), pushed to upstream casehubio/connectors main.

## Immediate Next Step

Both repos on `main`. #7 and #8 done. Pick from What's Next — #6 if qhorus#131
has landed; otherwise #9 or #10 (both independent of the Qhorus block).

## Cross-Module

**We're blocking:**
- `casehub-qhorus` — connectors#6 (Qhorus bridge: `InboundMessage` CDI event →
  `MessageService.dispatch()`) depends on `casehub-connectors-webhook` being published

**Blocked by:**
- `casehub-qhorus#131` — ChannelBackend SPI must land before connectors#6 can
  resolve `externalChannelRef` → Qhorus channel

## What's Left

- parent#97 — `casehub-connectors.md` deep-dive needs email inbound module, purpose update, "does not do inbound" correction · XS · Low
- parent#98 — `PLATFORM.md` Capability Ownership + Repository Map need updating for inbound · XS · Low
- parent#89 — still says "outbound-only" (carried from last session) · XS · Low

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| #6 | Qhorus bridge — `InboundMessage` CDI event → `MessageService.dispatch()` | M | Med | Blocked on qhorus#131 |
| #9 | Email inbound — IMAP IDLE for near-real-time delivery | M | Med | Deferred from #7 |
| #10 | Email inbound — attachment/binary content support | M | Med | Deferred from #7 |
| #2 | Slack `ChannelBackend` (outbound → Qhorus) | L | Med | Blocked on qhorus#131 |

## References

- `docs/specs/2026-05-29-email-inbound-connector-design.md` — email inbound spec (rev 4)
- `docs/specs/2026-05-29-inbound-connector-spi-design.md` — webhook inbound SPI spec
- `docs/adr/0001-inbound-connector-type-separation.md` — pull vs webhook type decision
- `docs/DESIGN.md` — updated with email-inbound module, metadata columns
- `email-inbound/` — new module; `EmailInboundConnector`, `EmailInboundAccountProvider`
- `blog/2026-05-29-mdp01-email-arrives-imap-inbound.md` — session diary entry
- Garden (Greenmail): GE-20260529-aa8445 (no deliver()), GE-20260529-72f189 (port conflict), GE-20260529-dbea23 (SMTP adds Message-ID), GE-20260529-4691e8 (appendMessage technique), GE-20260529-59d35a (getGreenMail protected)
- Garden (Jakarta Mail): GE-20260529-a488bf (MimeMessage saveChanges required before isMimeType)
- Protocols: PP-20260529-d4bec0 (@ConfigProperty vs Preferences for static creds), PP-20260529-8e5948 (connectorId = type not account), PP-20260529-ab32a9 (sanitize XFF before logging)
