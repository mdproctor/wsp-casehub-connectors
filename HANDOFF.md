# Handoff — connectors

## Last Session

Both branches fully closed. connectors#6 squashed + merged to casehubio/connectors main.
qhorus#219 squashed (9→1 commit), PR opened as **casehubio/qhorus#222** — merge pending.
The PR has a rebase conflict in pre-existing branch commits (ReactiveChannelStore,
settings.local.json diverged from upstream/main); the qhorus team owns the resolution.

## Immediate Next Step

Monitor casehubio/qhorus#222 — once merged, ConnectorChannelBackend ships. No action needed
from the connectors side unless the qhorus team needs help resolving the rebase conflict.

## Cross-Module

**We're blocking:**
- `casehub-qhorus` — casehubio/qhorus#222 merge still pending

## What's Left

- parent#108 — casehub-qhorus.md deep-dive needs connector-backend module · XS · Low
- parent#109 — casehub-connectors.md fireAsync contract note · XS · Low

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| qhorus#9 | Email inbound — IMAP IDLE for near-real-time delivery | M | Med | Deferred from #7 |
| qhorus#10 | Email inbound — attachment/binary content support | M | Med | Deferred from #7 |
| qhorus#2 | Slack ChannelBackend (outbound → Qhorus) | L | Med | No longer blocked |

## References

*Unchanged — `git show HEAD~1:HANDOFF.md`*
