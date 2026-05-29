# Handoff — connectors

## Last Session

Designed and implemented the full inbound connector SPI (issue #4). Added
`InboundConnector` (pull-based lifecycle), `WebhookInboundConnector` (standalone,
no lifecycle), `InboundConnectorService` CDI event bus, sealed `WebhookResult` type,
and a new `casehub-connectors-webhook` Maven module with `WebhookRouter` + four
concrete connectors (Slack, Teams, WhatsApp, Twilio SMS). 81 tests. Two code
review rounds (21 findings total, all resolved). Branch closed, issue #4 closed,
pushed to `origin/main`.

## Immediate Next Step

Both repos are on `main`. No outstanding work on this repo. Pick from What's Next.

## Cross-Module

**We're blocking:**
- `casehub-qhorus` — connectors#6 (Qhorus bridge: `InboundMessage` CDI event →
  `MessageService.dispatch()`) depends on `casehub-connectors-webhook` being published

**Blocked by:**
- `casehub-qhorus#131` — ChannelBackend SPI must land before connectors#6 (Qhorus
  bridge) can resolve `externalChannelRef` → Qhorus channel

## What's Left

- `casehub-connectors.md` deep-dive and `PLATFORM.md` capability table still say
  "outbound-only" — parent#89 filed, not yet actioned · XS · Low

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| #6 | Qhorus bridge — `InboundMessage` CDI event → `MessageService.dispatch()` | M | Med | Blocked on qhorus#131 for channel resolution |
| #7 | Email inbound — IMAP/SMTP polling, pull-based lifecycle | M | Low | Independent of qhorus block; uses existing `email` module |
| #8 | v1 polish — double JSON parse in Slack, unused `metadata`, structured security log | S | Low | connectors#8 filed |
| #2 | Slack `ChannelBackend` (outbound → Qhorus) | L | Med | Blocked on qhorus#131 |

## References

- `docs/specs/2026-05-29-inbound-connector-spi-design.md` — final spec (rev 3)
- `docs/adr/0001-inbound-connector-type-separation.md` — pull vs webhook type decision
- `docs/DESIGN.md` — updated with inbound SPI, module structure, data model
- `webhook/src/main/java/io/casehub/connectors/webhook/` — four concrete connectors
- Garden: GE-20260529-ab148d (CDI @Inject on multiple constructors), GE-20260529-c1e783 (dual-constructor CDI pattern), GE-20260529-709049 (sealed nested records), GE-20260529-d57945 (JAX-RS header case)
- Protocols: PP-20260529-7b94ab (inbound connector type separation), PP-20260529-b7765c (webhook HMAC constant-time)
