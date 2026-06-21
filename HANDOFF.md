# Handoff — connectors

## Last Session

CloudEvent adapter consistency work across three repos. Implemented connectors#20
(ConnectorCloudEventAdapter in new cloud-events submodule), fixed iot#26
(IoTCloudEventAdapter — 4 fixes), fixed qhorus#293 (fireAsync + 14 test migrations).
InboundMessage gained connectorType + tenancyId fields. Canonical 6-rule adapter
pattern captured in garden (GE-20260621-629712). All three issues closed.

## Immediate Next Step

Check open issues: `gh issue list --repo casehubio/connectors`

## What's Left

- Delete overdue closed branches (issue-4, 6, 7, 9, 12) — past 14-day hold · XS · Low

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| (idea) | Quarkus demo chat service — Slack-like, no Docker, for demos | M | Med | See IDEAS.md |

## References

| What | Path |
|------|------|
| Spec | `docs/specs/2026-06-21-cloudevent-adapter-consistency-design.md` |
| Garden entry | `GE-20260621-629712` — canonical CloudEvent adapter pattern |
| Blog | `blog/2026-06-21-mdp13-three-adapters-one-pattern.md` |
| IoT commit | `iot@a1c130e` — Closes iot#26 |
| Qhorus commit | `qhorus@4e9869b` — Closes qhorus#293 |
