# Handoff — connectors

## Last Session

Implemented connectors#6 (fireAsync ICS change) and qhorus#219 (ConnectorChannelBackend bridge).
connectors#6 is **closed** — squashed to 1 commit, pushed to casehubio/connectors main.
qhorus#219 branch is **complete but not yet work-end'd** — 4 commits ahead of qhorus main, including
two post-review linter commits: `ef5ca8e` adds `ChannelBindingStore.findAll()` and `7a36803` refactors
the mapper to Option B (caller supplies `Optional<ChannelConnectorBinding>`, `QhorusMcpToolsBase` owns lookup).

## Immediate Next Step

Run `work-end` on the qhorus branch `issue-219-connector-channel-backend` from
`/Users/mdproctor/claude/casehub/qhorus`. Before that, verify the build still passes with the
Option B mapper refactoring — run `mvn test -pl runtime,connector-backend` to confirm.

## Cross-Module

**We're blocking:**
- `casehub-qhorus` — needs casehubio/qhorus#219 merged before `ConnectorChannelBackend` ships

**What's Left (qhorus branch before work-end):**
- Verify build passes with Option B mapper + `findAll()` · XS · Low
- parent#108 — casehub-qhorus.md deep-dive needs connector-backend module · XS · Low (filed)
- parent#109 — casehub-connectors.md fireAsync contract note · XS · Low (filed)
- Run work-end on qhorus branch (squash 4 commits → close qhorus#219) · S · Low

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| qhorus#9 | Email inbound — IMAP IDLE for near-real-time delivery | M | Med | Deferred from #7 |
| qhorus#10 | Email inbound — attachment/binary content support | M | Med | Deferred from #7 |
| qhorus#2 | Slack ChannelBackend (outbound → Qhorus) | L | Med | No longer blocked |

## References

- `specs/2026-05-29-connector-channel-backend-design.md` — design spec (in workspace + promoted to connectors/docs/specs/)
- `blog/2026-05-30-mdp02-inbound-message-bridge.md` — session diary entry
- Garden: GE-20260530-b68c00 (Maven .lastUpdated cached failure), GE-20260530-0178fd (Quarkus deployment JAR for @QuarkusTest)
- parent#108, parent#109 — peer-repo doc issues filed (deep-dive updates needed)
