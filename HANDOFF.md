# HANDOFF — casehub-connectors

## Last Session

Completed #273 — convenience overloads for `ComposeAttachment.hero()` and `thumbnail()`.

**#273 — hero()/thumbnail() convenience overloads:**
- Added 1-arg (title) and 2-arg (title, subtitle) overloads for both `hero()` and `thumbnail()`
- Each delegates to the existing 3-arg form — no new logic, same `CardBuilder`
- 8 new tests: happy path + null-title validation for all 4 overloads
- Squashed formatting fixup into feat commit before landing

**Blog revision:**
- Revised mdp26 "The Line Between Raw and Ready" — added "Progressive disclosure" section
- Covers the pattern of shipping the full-spec form first, then adding convenience overloads

**Issues filed and scheduled:**
- #276 — `Builder.preview()` convenience overload for title-only (XS/Low)
- #277 — `CardBuilder` buttons support for hero/thumbnail cards (S/Low)

## Branch State

All work is on main. No feature branches open. Published to origin.

## Queue

2 items scheduled:
- [ ] #276 — preview convenience overload (XS/Low)
- [ ] #277 — CardBuilder buttons support (S/Low)

## Immediate Next Step

Pick up #276 (XS/Low) — same progressive disclosure pattern applied to `Builder.preview()`.

## References

- `chat-teams/src/main/java/io/casehub/connectors/chat/teams/ComposeAttachment.java` — Builder + CardBuilder with all overloads
- `blog/2026-11-17-mdp26-the-line-between-raw-and-ready.md` — revised blog with progressive disclosure section
- #276 — filed: preview() title-only overload
- #277 — filed: CardBuilder buttons support
