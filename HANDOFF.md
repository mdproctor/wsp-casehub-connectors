# Handoff — connectors

## Session summary

**Landed this session:**
- **#345** — Deferred to platform#938 (LocalDate coercion). Closed.
- **#346** — Fixed ref speaker embedding (SHA-256, 16-dim [-1,1]). Closed.
- **#339** — Telegram Presence capability (getChatMember status mapping). Closed.
- **#340** — Telegram Commands capability (setMyCommands). Closed.

**On branch (not landed):**
- **#208** — `mobile-numbrrs` module: Numbrrs MobilePlatform provider. Sms (read-only) + NumberInfo + WebhookInboundConnector with HMAC-SHA256 verification. 15 tests. Code complete.

**Design in progress:**
- **#356** — Epic: ConnectorLifecycle SPI. 7 architectural decisions captured in `specs/issue-208-mobile-numbrrs/decisions.md`. Decisions cover: replace-and-unify strategy, YAML manifests, connector families, browser-automated OAuth, hybrid schema, MCP domain API layering.

**Also filed:** #354 (PlatformRegistration — subsumed into #356 Phase 4), #209 closed (deferred)

## Cross-repo state

| Repo | Branch | Status |
|------|--------|--------|
| connectors | issue-208-mobile-numbrrs | 1 commit (mobile-numbrrs module) |
| wsp-connectors | issue-208-mobile-numbrrs | decisions.md, pipeline.state |

## Next session

1. **Land #208** — the mobile-numbrrs module is code-complete, needs work-end
2. **Phase 1 spec for #356** — continue brainstorm from decisions.md. Key open questions: manifest schema detail, lifecycle manager internals, first three connector migrations (Numbrrs, Telegram, Signal)
3. **Signal testing** — once #208 lands, can test Signal registration using Numbrrs number via the automated flow from #356 Phase 4

## Notes

- platform#938 needs pickup in a platform session for LocalDate fix
- Telegram now 7 native capabilities (was 5)
- Playbook speaker name assertions blocked by voice-sherpa yaml-services.yaml override — ref unit tests verify discrimination directly
