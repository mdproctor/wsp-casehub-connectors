# HANDOFF — casehub-connectors

## Last Session

Designed and partially implemented MobilePlatform SPI (#206) — capability-based telephony for SMS, voice calls, voicemail, WhatsApp/iMessage.

**This session:**
- Brainstormed and captured 14 design decisions (instruction-based voice calls, ConversationHandler for agentic calls, two call modes, provider scope)
- Built mobile-spi (6 capabilities, Builder, NoOp, PlatformService — 20+ model records)
- Built mobile-ref (in-memory switchboard, canned conversations, seed data)
- Created ConnectorMobileApi (9 MCP tools), wired MobilePlatformService into ConnectorOperationsImpl
- Rewrote dinner booking playbook (5 domains, 6 steps — location + mobile + email + chat)
- 175 tests pass (15 spi + 16 ref + 144 graphql)

**Key decisions:** Voice calls are instruction-based at the SPI surface (D1). Two modes: provider-managed (fire-and-forget) and platform-managed (ConversationHandler with mid-call tool access) (D8). ConversationHandler callback at the SPI level; eidos AgentDescriptor in playbooks for best practice (D9). Sealed ConversationAction hierarchy (Hold/Transfer/HangUp).

## Immediate Next Step

**Batch 4: Dial Provider (mobile-dial).** REST client for SMS/messaging/numberInfo, self-hosted LLM WebSocket for platform-managed voice calls. Then Batch 5: Twilio + legacy absorption. Plan: `plans/2026-11-12-mobile-platform.md`.

## Cross-Module

- **Blocked by** — none
- **Blocks** — dinner booking playbook needs mobile-ref on classpath (already wired)

## References

- Spec: `specs/issue-206-mobile-platform/2026-11-12-mobile-platform-spi-design.md`
- Decisions: `specs/issue-206-mobile-platform/decisions.md`
- Plan: `plans/2026-11-12-mobile-platform.md`
- Blog: `blog/2026-11-12-mdp01-when-the-agent-picks-up-the-phone.md`
- Design brief: `docs/specs/issue-206-mobile-platform/design-brief.md` (in project repo)
