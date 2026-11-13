# HANDOFF — casehub-connectors

## Last Session

Continuing MobilePlatform SPI (#206) implementation — batches 1-5 complete.

**This session:**
- Built mobile-dial module (Batch 4) — Dial provider with REST client + self-hosted LLM WebSocket
- Built mobile-twilio module (Batch 5) — Twilio provider with REST client, TwilioSmsBridge, TwilioWhatsAppBridge
- Legacy absorption: deleted TwilioSmsConnector, WhatsAppConnector, TwilioSmsInboundConnector, WhatsAppInboundConnector
- Cleaned up ConnectorsBeans, ConnectorsAutoConfiguration, WebhookBeans, ConnectorMessagingApi
- Removed sendSms/sendWhatsapp from ConnectorMessagingApi (now via ConnectorMobileApi)
- Preserved connectorId continuity: "twilio-sms-inbound" and "whatsapp-inbound"
- 85 new tests across mobile-dial (53) and mobile-twilio (32), all passing

**Pre-existing failures (not from this session):**
- voice-pipeline-steps: TTS tests expect kokoro voice platform not available locally
- integration-tests: VoiceRoundTripTest same kokoro dependency
- playbooks: dinner booking tests from batch 3 need investigation

## Immediate Next Step

**Batch 6: Integration Tests + Documentation.** MobileRefIntegrationTest in integration-tests, CLAUDE.md module table update, consumer/contributor guide updates, deferred scope issue filing. Plan: `plans/2026-11-12-mobile-platform.md`.

## Cross-Module

- **Blocked by** — none
- **Blocks** — dinner booking playbook needs mobile-ref on classpath (already wired)

## References

- Spec: `specs/issue-206-mobile-platform/2026-11-12-mobile-platform-spi-design.md`
- Decisions: `specs/issue-206-mobile-platform/decisions.md`
- Plan: `plans/2026-11-12-mobile-platform.md`
- Blog: `blog/2026-11-12-mdp01-when-the-agent-picks-up-the-phone.md`
- Design brief: `docs/specs/issue-206-mobile-platform/design-brief.md` (in project repo)
