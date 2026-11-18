# HANDOFF — casehub-connectors

## Last Session

Landed 11 commits closing 12 issues on branch `issue-253-teams-replyToMessage-user-prefix`. Published to both remotes.

### What landed

| Issue | Change | Module |
|-------|--------|--------|
| #253 | Handle `user:` channel prefix in `replyToMessage` — resolve via conversation reference store or cold-start create | chat-teams |
| #293 | Skip `#` and `?` inside Graph `:/path:/` segments in `toUri` — `graphPathEnd()` finds the boundary | microsoft-graph-client |
| #295+#296 | Audit all 14 `@McpDomain` APIs for `@GeneratePlugin` coverage — only gap: `ConnectorChatApi.listChannels()` marked `exclude=true`; dead import removed from ConnectorTravelApi | graphql |
| #298 | `TranscriptNormaliser` utility for STT round-trip comparison — strips digit-grouping commas, collapses hyphens/spaces | connectors-api |
| #299 | Guard `ChannelManagement.delete`/`find` against `personal:` channels | chat-teams |
| #300 | Map Bot Framework errors to `ProviderApiException` hierarchy via `ProviderErrorMapping.mapStatusCode()` | chat-teams |
| #301 | Multi-pass loop for same-tag HTML nesting in `htmlToMarkdown` | chat-teams |
| #302 | `@Named("gmail-push")` qualifier on `JwtValidator` producer and injection | email-google-inbound |
| #304 | Attachment download: 10s timeout, streaming with 8MB size limit, Content-Length pre-check | chat-teams |
| #305 | HTTPS scheme enforcement in SSRF defense + InputStream leak fix on early return paths | chat-teams |
| #297 | Confirmed `ConnectorMessagingApi` has 3 methods (not 5 as #264 stated) — no code change | — |

### Follow-up issues filed

| # | Title | Scale | Complexity |
|---|-------|-------|------------|
| #312 | Guard `user:` channel in listMembers, messageHistory, reactions, memberManagement | XS | Low |
| #313 | Extract conversation resolution from sendProactiveMessage/replyToUserChannel | XS | Low |
| #314 | BotFrameworkClient.checkResponse drops URL from diagnostics | XS | Low |
| #315 | Add `contains-normalised:` assertion to YAML playbook test framework (platform-side) | S | Med |

Blog entry: `blog/2026-11-18-mdp05-the-same-gap-twelve-times.md` — defensive completeness as systematic practice, fix chains surfacing the same latent gap in adjacent operations.

### Code review findings addressed

- InputStream not closed on non-200 or oversized responses — fixed (moved checks inside try-with-resources)
- Import ordering in BotFrameworkClient — fixed
- Test method typo `toUriHandsHash` → `toUriHandlesHash` — fixed
- Duplicated conversation resolution logic — filed as #313
- `user:` channel guards missing from remaining operations — filed as #312
- `checkResponse` drops URL context — filed as #314

## Immediate Next Step

Queue has 5 remaining items:

| # | Title | Scale | Complexity |
|---|-------|-------|------------|
| #308 | voice-ref-mark-timing-support | XS | Low |
| #307 | VITS-guarded-exact-mark-resolution | M | Med |
| #312 | guard-user-channel-remaining-operations | XS | Low |
| #313 | extract-conversation-resolution | XS | Low |
| #314 | checkResponse-url-diagnostics | XS | Low |

#307 blocks casehub-agency#457 (gesture timing D4). #312-#314 are review follow-ups from this session.

## Context for Next Session

- All 12 issues were landed on a single branch. The queue was pruned of 49 completed items from earlier sessions — now only 5 items remain.
- `replyToUserChannel` duplicates conversation resolution logic from `sendProactiveMessage` — both resolve AAD → conversation via store + cold-start. Filed as #313 for extraction.
- `user:` channel guards were added to `replyToMessage` (#253), `delete`/`find` (#299), but `listMembers`, `listMessageHistory`, reactions, and member management still only guard `personal:` — filed as #312.
- `TranscriptNormaliser` is in `connectors-api` for programmatic use. YAML playbook `contains-normalised:` requires a platform-side change — filed as #315 on connectors but scope is platform.
- `DiscordClient` and `BotFrameworkClient` now have matching patterns for attachment download (timeout, streaming size limit, SSRF host+scheme check). Teams added `InterruptedException` handling that Discord already had.
