# Handoff — connectors

## Last Session

Closed #23 — moved ConnectorCloudEventAdapter from separate cloud-events
submodule into core, renamed to ConnectorsCloudEventAdapter. Direct
cloudevents-core dependency (not via casehub-platform-api) preserves
zero-casehubio-dep invariant. Discovered quarkus-jackson needed instead
of raw jackson-databind for CDI ObjectMapper producer in downstream
@QuarkusTest modules. ARC42STORIES §1/§2 rewritten to name the real
invariant ("Lightweight foundation", not "Zero-dep core").

## Immediate Next Step

No open issues. Check `gh issue list --repo casehubio/connectors` or pick from ideas.

## What's Left

- Delete overdue closed branches (issue-4, 6, 7, 9, 12) — past 14-day hold · XS · Low

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| (idea) | Quarkus demo chat service — Slack-like, no Docker, for demos | M | Med | See IDEAS.md |

## References

*Unchanged — `git show HEAD~1:HANDOFF.md`*
