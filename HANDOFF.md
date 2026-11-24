# Handoff — connectors

## Session summary

**Landing this session:**
- **#208** — `mobile-numbrrs` module: Numbrrs MobilePlatform provider. Sms (read-only) + NumberInfo + NumbrrsInboundConnector (HMAC-SHA256 webhook, sms.received + voice.received events). Added InboundConnectorTypes.VOICE constant. 17 tests. Review cleanup done. Consumer guide, contributor guide, CLAUDE.md updated. Blog written.

**Previously landed (prior session):**
- **#345** — Deferred to platform#938 (LocalDate coercion). Closed.
- **#346** — Fixed ref speaker embedding (SHA-256, 16-dim [-1,1]). Closed.
- **#339** — Telegram Presence capability (getChatMember status mapping). Closed.
- **#340** — Telegram Commands capability (setMyCommands). Closed.

**Design in progress:**
- **#356** — Epic: ConnectorLifecycle SPI. 7 architectural decisions captured in `specs/issue-208-mobile-numbrrs/decisions.md`. Decisions cover: replace-and-unify strategy, YAML manifests, connector families, browser-automated OAuth, hybrid schema, MCP domain API layering.

## Cross-repo state

| Repo | Branch | Status |
|------|--------|--------|
| connectors | issue-208-mobile-numbrrs | 1 commit (squashed), ready to land |
| wsp-connectors | issue-208-mobile-numbrrs | 1 commit (squashed), decisions.md + blog + handoff |

## Next session

1. **Phase 1 spec for #356** — continue brainstorm from decisions.md. Key open questions: manifest schema detail, lifecycle manager internals, first three connector migrations (Numbrrs, Telegram, Signal)
2. **Signal testing** — once #208 lands, test Signal registration using Numbrrs number

## Notes

- platform#938 needs pickup in a platform session for LocalDate fix
- Telegram now 7 native capabilities (was 5)
- Playbook speaker name assertions blocked by voice-sherpa yaml-services.yaml override — ref unit tests verify discrimination directly
