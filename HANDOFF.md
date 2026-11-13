# HANDOFF — casehub-connectors

## Last Session

Continuing MobilePlatform SPI (#206) implementation — batches 1-4 complete.

**This session:**
- Built mobile-dial module (Batch 4) — Dial provider for MobilePlatform SPI
- DialClient: REST client for Dial API (SMS, messaging, voice calls, number queries) with Bearer auth and ProviderErrorMapping error handling
- DialWebSocket: self-hosted LLM WebSocket client for platform-managed voice calls via ConversationHandler — handles response_required/response protocol, sealed ConversationAction dispatch
- DialMobilePlatform: wires DialClient to 5 capabilities (Sms, VoiceCall, VoiceCallControl, Messaging, NumberInfo) via Builder. No Voicemail — Dial doesn't support it natively
- DialBeans: CDI producer using StaticCredentialStore for credential resolution
- 53 new tests (6 DialClient JSON mapping + 47 DialMobilePlatform mapping/wiring), all passing
- Full build chain (mobile-spi → mobile-ref → mobile-dial → graphql → connectors-spring) green

**Pre-existing failures (not from this session):**
- voice-pipeline-steps: TTS tests expect kokoro voice platform not available locally
- integration-tests: VoiceRoundTripTest same kokoro dependency
- playbooks: dinner booking tests from batch 3 need investigation

## Immediate Next Step

**Batch 5: Twilio Provider + Legacy Absorption (mobile-twilio).** TwilioClient, TwilioMediaStreams (WebSocket + VoicePlatform STT/TTS for platform-managed calls), TwilioMobilePlatform, legacy TwilioSmsConnector/WhatsAppInboundConnector absorption. Then Batch 6: Integration tests and docs. Plan: `plans/2026-11-12-mobile-platform.md`.

## Cross-Module

- **Blocked by** — none
- **Blocks** — dinner booking playbook needs mobile-ref on classpath (already wired)

## References

- Spec: `specs/issue-206-mobile-platform/2026-11-12-mobile-platform-spi-design.md`
- Decisions: `specs/issue-206-mobile-platform/decisions.md`
- Plan: `plans/2026-11-12-mobile-platform.md`
- Blog: `blog/2026-11-12-mdp01-when-the-agent-picks-up-the-phone.md`
- Design brief: `docs/specs/issue-206-mobile-platform/design-brief.md` (in project repo)
