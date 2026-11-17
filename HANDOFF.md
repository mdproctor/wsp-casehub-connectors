# HANDOFF — casehub-connectors

## Last Session

Completed #257 (Teams bot manifest and app registration for messaging extensions). Created a manifest template with all three extension types (search, action, link unfurling) and a standalone setup guide covering Azure bot registration prerequisites, InvokeHandler wiring, commandId routing with worked example, and Quarkus config. Post-close review caught an incorrect webhook endpoint URL (`/webhooks/teams` → `/connectors/teams-inbound/webhook`) — fixed on main. Filed #265, #266, #267 for loose ends.

## Immediate Next Step

`work start 259` — messaging-extension-preview-card-model. Same Teams messaging extensions campaign, same module.

## References

- `docs/guides/teams-messaging-extensions-setup.md` — the setup guide
- `chat-teams/src/main/resources/manifest-template.json` — the manifest template
- `docs/blog/2026-11-17-mdp25-the-dispatch-that-didnt-change.md` — diary entry with two-level dispatch diagram
- `.queue` — 2 items remaining (#259, #258)
