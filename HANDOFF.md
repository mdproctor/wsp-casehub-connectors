# HANDOFF — casehub-connectors

## Last Session

Completed 4 issues (#273, #276, #277, #279) — progressive API surface for `ComposeAttachment` builder.

**#273 — hero()/thumbnail() convenience overloads:**
- 1-arg (title) and 2-arg (title, subtitle) overloads for both factory methods

**#276 — preview(title) convenience overload:**
- `Builder.preview(String title)` delegates to `preview(title, null)`

**#277 — CardBuilder buttons support:**
- `button(String type, String title, String value)` — appending, protocol vocabulary as strings

**#279 — CardBuilder tap action:**
- `tap(String type, String title, String value)` — single object, last-call-wins (replace not append)

**Blog:**
- Revised mdp26 "The Line Between Raw and Ready" three times across the session:
  - "Progressive disclosure" section (ship full-spec first, peel off common shapes)
  - "Buttons and the boundary within a call" section with SVG diagram
  - `tap()` paragraph — single vs array, the protocol dictates the shape

## Branch State

All work on main. No feature branches open. Published to origin.

## Queue

Drained — no items scheduled.

## Open Issues (not scheduled)

- #274 — DRY card content JSON builder (S/Low)
- #275 — CardBuilder round-trip test (XS/Low)

## References

- `chat-teams/src/main/java/io/casehub/connectors/chat/teams/ComposeAttachment.java` — complete with preview/button/tap/overloads
- `blog/2026-11-17-mdp26-the-line-between-raw-and-ready.md` — session blog
- `blog/images/compose-attachment-boundary.svg` — attachment boundary diagram
- `blog/images/compose-attachment-buttons.svg` — buttons boundary diagram
