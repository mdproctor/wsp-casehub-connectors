# HANDOFF — casehub-connectors

## Last Session

Completed #276 and #277 — `Builder.preview()` convenience overload and `CardBuilder` buttons support. Also earlier in the session: #273 convenience overloads for `hero()` and `thumbnail()`.

**#273 — hero()/thumbnail() convenience overloads (landed earlier):**
- 1-arg (title) and 2-arg (title, subtitle) overloads for both `hero()` and `thumbnail()`
- Each delegates to the existing 3-arg form

**#276 — preview(title) convenience overload:**
- `Builder.preview(String title)` delegates to `preview(title, null)`
- Completes the progressive disclosure pattern across all three entry points

**#277 — CardBuilder buttons support:**
- `button(String type, String title, String value)` on `CardBuilder`
- Buttons array rendered in JSON only when buttons are added
- All three args required (non-null) — type is protocol vocabulary (string, not enum for forward compatibility)
- Lazy `ArrayList` allocation — no cost when buttons aren't used

**Blog revision:**
- Revised mdp26 "The Line Between Raw and Ready" twice:
  - Added "Progressive disclosure" section (#273)
  - Added "Buttons and the boundary within a call" section (#277) with SVG diagram
- Key design insight: the protocol boundary can run through a single method call — shape is protocol, values are caller-chosen

**Issues filed and scheduled:**
- #279 — `CardBuilder` tap action (XS/Low) — same pattern as button() but single object not array

**Code review:** 0 CRITICAL, 0 WARNING across both issues. Minor formatting fixes applied.

## Branch State

All work on main. No feature branches open. Published to origin.

## Queue

1 item scheduled:
- [ ] #279 — CardBuilder tap action (XS/Low)

Also open but not scheduled:
- #274 — DRY card content JSON builder (S/Low)
- #275 — CardBuilder round-trip test (XS/Low)

## Immediate Next Step

Pick up #279 (XS/Low) — same `button()` pattern applied to the card-level `tap` property.

## References

- `chat-teams/src/main/java/io/casehub/connectors/chat/teams/ComposeAttachment.java` — Builder + CardBuilder with preview/button/overloads
- `blog/2026-11-17-mdp26-the-line-between-raw-and-ready.md` — revised blog with progressive disclosure + buttons sections
- `blog/images/compose-attachment-buttons.svg` — buttons boundary diagram
