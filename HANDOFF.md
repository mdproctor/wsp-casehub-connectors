# Handoff — connectors

## Last Session

Closed branch `issue-89-notification-calendar` — three issues landed:
- **#89** config-based `DestinationResolver` fallback in `NotificationBridgeStartup`
- **#91** `DigestFormatter` CDI SPI (email HTML, SMS, WhatsApp) + `EmailConnector` format=html attribute
- **#88** `CalendarPlatform` SPI (`calendar-spi`, `calendar-ref`, `calendar-google`) + `CalendarMcpTool` (6 tools)

Design spec adversarially reviewed (5 rounds, 20 issues, all resolved).
Pushed directly to upstream/main. 9 commits after squash.

## Immediate Next Step

Pick up #90 (per-tenant destination deduplication) — unblocks Slack/Teams
notification bridging. Or #45 (Teams ChatPlatform) if messaging breadth
is the priority.

## What's Left

- Delete overdue closed branches (issue-4, 6, 7, 9, 12, 39, 86, 89) · XS · Low
- 14 unrecovered artifacts on closed workspace branches (specs, blogs) · S · Low

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| #90 | Per-tenant destination deduplication | M | Med | Unblocks Slack/Teams notification bridging |
| #45 | Teams ChatPlatform implementation | M | Med | Requires Teams Bot API client |
| #58 | Responsive layout primitives for pages-runtime | L | High | Cross-module design needed |
| #32 | Discord slash commands and interactions | M | Med | — |

## References

*Unchanged — `git show HEAD~1:HANDOFF.md`*
