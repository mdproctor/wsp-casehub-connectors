# Handoff — connectors

## Session summary

- **#345 — LocalDate step coercion**: Deferred to platform repo. Filed casehubio/platform#938. No connectors changes.
- **#346 — Voice-ref speaker discrimination**: Fixed. SHA-256 replaces hashCode in `RefSpeakerEmbeddingExtractor` (16-dim [-1,1]). 4 unit tests. Closed.
- **#339 — Telegram Presence**: Implemented `TelegramPresence` inner class — maps `getChatMember` status to PresenceStatus. 6 WireMock tests.
- **#340 — Telegram Commands**: Implemented `TelegramCommands` inner class — delegates to `setMyCommands`. 1 WireMock test. Telegram now 7 native capabilities.

## Cross-repo state

| Repo | Branch | Status |
|------|--------|--------|
| connectors | issue-340-telegram-commands | 1 commit ahead of main (squashed) |
| wsp-connectors | main | queue updates, handoff |

## Queue

All items done. Queue empty after landing.

## Notes

- platform#938 needs pickup in a platform session for LocalDate fix
