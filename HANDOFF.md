# HANDOFF — casehub-connectors

## Last Session

Hardened ComposeAttachment and invoke dispatch chain — four XS/S issues (#269-#272), all committed directly to main.

**#269 — preview() null validation:**
- `Objects.requireNonNull(title, "title")` on both preview() overloads
- Consistent with existing `contentJson` constructor guard — fail-fast at call site

**#270 — hero() and thumbnail() factory methods:**
- New `CardBuilder` inner class for structured Hero/Thumbnail cards
- `ComposeAttachment.hero(title, sub, img).build()` and `.thumbnail()`
- Same null guard on title across all three entry points

**#271 — @QuarkusTest for InvokeHandler CDI dispatch:**
- Proves CDI discovery of `@ApplicationScoped` handlers via `@All` injection
- Full chain: CDI → InvokeService.dispatch() → handler → InvokeResponseSerializer.toJson()
- Required StubServiceConnectionProvider + application.properties for TeamsBeans deps
- 5 tests: CDI discovery, full chain serialization, multiple handlers, unknown name, unknown serialization

**#272 — RefSearchHandler query filtering:**
- Case-insensitive substring filter on `queryText` — demonstrates the query→filter→response pattern
- Blank or missing query returns all results (backward compatible)
- 5 new tests covering blank, missing, matching, case-insensitive, and no-match scenarios

**Other session work:**
- Blog: mdp27 "The Gap Between Works and Trustworthy" with SVG diagram
- Code review: clean — no bugs, no thread safety issues, no dead code
- Filed 3 new issues: #273 (convenience overloads), #274 (DRY card content builder), #275 (CardBuilder round-trip test)
- Closed all 4 issues on GitHub

## Branch State

All work is on main. No feature branches open. Queue is drained.

5 unpublished commits on main (ahead of origin/main):
- `3b880faa` fix(#269)
- `f036b186` feat(#270)
- `3114e913` test(#271)
- `1f25725d` feat(#272)
- `ab7f94bd` docs(blog)

## Immediate Next Step

Publish to remote. The 3 newly filed issues (#273-#275) are all XS/Low enhancement follow-ups — not urgent.

## References

- `chat-teams/src/main/java/io/casehub/connectors/chat/teams/ComposeAttachment.java` — Builder + CardBuilder
- `chat-teams/src/main/java/io/casehub/connectors/chat/teams/ref/RefSearchHandler.java` — query filtering
- `chat-teams/src/test/java/io/casehub/connectors/chat/teams/InvokeServiceCdiTest.java` — CDI integration test
- `blog/2026-11-17-mdp27-the-gap-between-works-and-trustworthy.md` — session blog
- #273 — filed: convenience overloads for hero()/thumbnail()
- #274 — filed: DRY card content JSON builder
- #275 — filed: CardBuilder round-trip test
