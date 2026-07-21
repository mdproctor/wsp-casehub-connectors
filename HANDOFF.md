# Handoff — connectors

## Last Session

Closed branch `issue-86-notification-delivery-bridge` — new
`notification-bridge` module bridges platform `NotificationDeliverer`
SPI to `Connector` SPI. Breaking change: `Connector.send()` → `boolean`.
`channelType()` default method for channel type mapping. `DestinationResolver`
SPI added to `casehub-platform-api`. PR #87 to upstream. Closes #86.

Cross-repo: platform branch `issue-86-destination-resolver` adds
`DestinationResolver` + `DeliveryChannels.WHATSAPP` to platform-api.

## Immediate Next Step

Merge platform `issue-86-destination-resolver` before connectors PR #87.
Then pick from What's Next.

## What's Left

- Delete overdue closed branches (issue-4, 6, 7, 9, 12, 39, 86) · XS · Low
- Merge platform branch `issue-86-destination-resolver` · XS · Low

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| #45 | Teams ChatPlatform implementation | M | Med | Requires Teams Bot API client |
| #58 | Responsive layout primitives for pages-runtime | L | High | Cross-module design needed |
| #32 | Discord slash commands and interactions | M | Med | — |

## References

*Unchanged — `git show HEAD~1:HANDOFF.md`*
