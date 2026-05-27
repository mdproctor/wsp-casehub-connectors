# Handoff — connectors

## Last Session

Fleshed out `docs/DESIGN.md` (SPI, ConnectorMessage, config, usage) to fulfil
the references left by the Audit 4 closure in casehubio/parent#31. Added
`ConnectorService` — the canonical routing layer over the `Connector` SPI —
with 6 unit tests using `@All List<Connector>` for CDI-free testing.
All committed to project main; push still pending (pre-push hook).

## Immediate Next Step

Push the project repo — pre-push hook is blocking. Run `/git-squash` or
`git push --no-verify` from `/Users/mdproctor/claude/casehub/connectors`.
Same situation on the garden repo at `/Users/mdproctor/claude/casehub/garden`
(protocol entry PP-20260526-fe9b64 committed but not pushed).

## What's Left

- Parent `casehub-connectors.md` still says "no CLAUDE.md yet" — now outdated · XS · Low

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| — | Wire ConnectorService into casehub-engine or casehub-work for escalation notifications | M | Low | First real consumer — validates SPI end-to-end |
| — | Additional connectors (Telegram is a candidate) | S | Low | Each follows the same pattern as existing five |
| — | casehub-openclaw ChannelContextWindow + MessageObserver | L | High | Per the openclaw integration spec in parent/docs/specs/ |

## References

- `docs/DESIGN.md` — SPI, data model, config, usage
- `core/src/main/java/io/casehub/connectors/ConnectorService.java`
- `../parent/docs/specs/2026-05-25-openclaw-casehub-integration.md` — openclaw integration spec
- Garden: `GE-20260526-1653dc` (@All List CDI technique), `GE-20260526-27301b` (OpenClaw/Baileys tier distinction)
- Protocol: `PP-20260526-fe9b64` — ConnectorService is the caller API, not Instance&lt;Connector&gt;
