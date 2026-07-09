# Handoff — connectors

## Last Session

Designed and implemented a qhorus-native chat UI component family (Phase 1).
Brainstormed the full conversation model (Space→Channel→Topic→Thread) after
researching 11 chat platforms and 6 agent frameworks. Ran adversarial design
review (7 rounds, 27 issues). Implemented 9 tasks via SDD: 4 primitives
(qhorus-message, reaction-bar, thread, message-input), 3 composites
(channel-feed, channel-nav, member-panel), adapter + workbench, atomic swap
deleting old code. 182 tests after comprehensive audit. Final code review
(2 rounds, 24 issues). Filed qhorus epic #328 with 6 child issues (#329–#334)
for conversation model enrichments. Filed 9 connectors issues (#62–#70) for
Phases 2–5.

## Immediate Next Step

Manual browser verification of the new UI. Start the dev server and navigate
to `http://localhost:8090/src/index.html`. Chrome MCP is now installed — use
it for automated browser testing. Verify: login gate, channel list, message
send/receive, reactions, flat/threaded toggle, member panel.

Then `work-end` to close branch `issue-61-qhorus-chat-ui` / #61.

## Cross-Module

**Qhorus can work in parallel:**
- Epic qhorus#328 with 6 child issues (#329–#334)
- Priority: #329 (Topic) and #330 (Reactions) — highest impact, independent

## What's Left

- Browser verification of new UI (Chrome MCP available) · S · Low
- #53 ARC42STORIES.MD sync — needs chat UI architecture additions · M · Med
- #70 Tech debt — a11y mixins, thread root selection, reaction perf · S · Med
- Delete overdue closed branches (issue-4, 6, 7, 9, 12) · XS · Low

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| #62 | Dockable contextual panels (Phase 2) | M | Med | artifact, task, correlation panels |
| #63 | Progressive disclosure (Phase 2) | S | Med | |
| #64 | Emoji reaction palette (Phase 3) | S | Med | Needs qhorus#330 |
| #65 | Rich artefact references (Phase 3) | M | Med | Needs qhorus#331 |
| #66 | Topic navigator (Phase 4) | M | Med | Needs qhorus#329 |
| #69 | Claudony integration (Phase 5) | L | High | Needs Phase 2 min |

## References

| Doc | Path |
|-----|------|
| Design spec | `specs/2026-07-07-qhorus-chat-ui-design.md` |
| Research | `specs/2026-07-07-conversation-model-research.md` |
| Plan | `plans/2026-07-07-qhorus-chat-ui-phase1.md` |
| Qhorus epic | casehubio/qhorus#328 |
