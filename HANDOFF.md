# Handoff — connectors

## Last Session

Issues #9 (IMAP IDLE) and #10 (binary attachments) designed, implemented, reviewed,
and shipped to casehubio/connectors main in two squashed commits. Branch
`issue-9-imap-idle-and-attachments` closed. Both issues closed on GitHub.

## Immediate Next Step

Fix IDLE test flakiness — casehubio/connectors#12. Root cause: 3-second receive
timeout in `EmailInboundConnectorTest.receive()` is too tight under JVM cold-start
load. Start by bumping to 5 seconds and re-running; if still flaky, switch to
Awaitility.

## Cross-Module

*Unchanged — `git show HEAD~1:HANDOFF.md`*

## What's Left

- connectors#12 — fix IDLE test timing flakiness (receive timeout too tight) · XS · Low
- parent#108 — casehub-qhorus.md deep-dive needs connector-backend module · XS · Low
- parent#109 — casehub-connectors.md fireAsync contract note · XS · Low

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| connectors#12 | Fix IDLE test flakiness — bump receive timeout or switch to Awaitility | XS | Low | Filed this session |
| connectors#2 | Slack ChannelBackend (outbound → Qhorus) | L | Med | No longer blocked |
| qhorus#9 | Email inbound — IMAP IDLE (near-real-time) | — | — | Closed this session |
| qhorus#10 | Email inbound — attachment/binary support | — | — | Closed this session |

## References

- Blog: `blog/2026-05-31-mdp03-imap-idle-and-attachments.md`
- ADR: `adr/0002-imap-idle-replaces-polling.md`
- Garden: GE-20260531-c41c7f (DCH gotcha), GE-20260531-2ec49a (idle() param), GE-20260531-68222f (exec amend technique)
- Protocol: PP-20260531-32efe8 (numeric metadata always present)
