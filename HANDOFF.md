# Handoff — connectors

## Last Session

ARC42STORIES.MD fully reconstituted from DESIGN.md, ADRs 0001–0003, blog
mdp01–mdp06, design/JOURNAL.md, and closed issues #1–#12. All 🔲 sections
filled: §3 C4Context, §4 layer taxonomy + chapter sequencing, §5 C4Container,
§6 four runtime scenarios, §8 three inline anti-patterns, §9 C1/C2/C3 with
C4Component diagrams + §9.4 layer×chapter matrix, §10–§12, §13 expanded
glossary. Closes #13. Project main (8 unpushed commits from last session's
MCP work) pushed during epic hygiene.

## Immediate Next Step

Run `work-end` for branch `issue-13-arc42stories` — squash the workspace
commits and merge to workspace main. Then start connectors#14 (convert
`singlePlainTextMessage_deliveredWithCorrectFields` to `deliverDirect()`).

## Cross-Module

**We're blocking** (Qhorus needs this before bridge can land):
- `qhorus` — qhorus#249: `ConnectorMeshBridge` impl in `connector-backend`. SPI ships in `core`, no-op default present. · S · Med

**Peer docs outstanding:**
- parent#168 — sync `casehub-connectors.md` deep-dive + PLATFORM.md for `mcp` module and `ConnectorMeshBridge` SPI. · XS · Low

## What's Left

- `issue-13-arc42stories` branch — needs `work-end` (squash + merge to main) · XS · Low

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| connectors#14 | Convert `singlePlainTextMessage` test to `deliverDirect()` | XS | Low | Quick win before next feature |
| connectors#2 | Slack ChannelBackend (outbound → Qhorus) | L | Med | No longer blocked |

## References

- ARC42STORIES.MD: workspace root
- Blog: `blog/2026-06-04-mdp07-what-design-md-doesnt-know.md`
- ADRs: `docs/adr/0001` – `0003`
- Issues closed: connectors#13
