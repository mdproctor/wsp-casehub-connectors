# Design Journal — issue-188-real-time-inbound-channels

## 2026-10-30 — Design session: scope reframing, playbook data pipelines, VoicePlatform SPI

**Context:** Epic #188 originally described three new inbound channels (WhatsApp, Gmail push, voice messages). Context gathering revealed WhatsApp inbound and email inbound already exist — the real gaps are media download, Gmail push, and the voice SPI.

**Key decisions:**
- D3: Playbook orchestration with streaming steps for the audio processing pipeline. The platform's step engine (originally for agent/human workflows) models data pipelines naturally when step types declare typed I/O contracts. Filed platform#680 for streaming step support — it landed within the session.
- D5: Gmail push-only via WebhookInboundConnector. IMAP IDLE already handles pull. No need for a Pub/Sub pull subscription.
- D6: VoicePlatform SPI scoped to STT + TTS only. The decision review (3 rounds, 15 issues) correctly identified that denoising/VAD/speaker-ID/lip-sync are processing capabilities, not channel interfaces. They violate connectors' "no domain knowledge" rule.
- D10: SPI in connectors, implementation stays in agency for now. Full migration deferred to #187 — native build toolchain (C++, CMake, 5 OS/arch targets) isn't in connectors CI.
- D11 (surfaced by review): Voice is a delivery channel, not a processing toolkit.
- D12 (surfaced by review): Playbook YAML lives in consuming applications, not connectors.

**Architecture:** 4 new modules (voice-spi, voice-ref, email-google-inbound, voice-pipeline-steps), 1 enhanced (webhook — WhatsApp media download via DeferredDelivery), 1 cross-repo adapter (agency/speech-sherpa implements VoicePlatform SPI).

**Plan:** 4 batches, 7 tasks. Ready for execution.
