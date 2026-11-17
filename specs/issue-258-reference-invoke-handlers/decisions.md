## D1: Module placement — directly in chat-teams

**Choice:** Reference handlers go in `chat-teams` module as public classes
**Alternatives:**
- New `chat-teams-ref` module — follows ref module pattern but those exist for cross-platform SPIs; InvokeHandler is Teams-only, a module for 4 handlers is overhead
- In `chat-teams` test sources — not reusable by downstream consumers or demos
**Rationale:** InvokeHandler is a Teams-specific SPI, not cross-platform like ChatPlatform or BankPlatform. Consumers who need invoke handlers already depend on `chat-teams`. No need for a separate ref module.
**Trade-offs:** No separate module for selective dependency — all handlers ship with `chat-teams`. Acceptable since they're lightweight with no additional dependencies.
**Sources:** InvokeHandler.java, chat-ref/pom.xml (depends on chat-spi not chat-teams), issue #258
**Exploration:** quick
**Status:** captured

## D2: Handler granularity — one class per invoke name

**Choice:** Four handler classes: RefSearchHandler, RefLinkUnfurlHandler, RefFetchTaskHandler, RefSubmitActionHandler
**Alternatives:**
- Combine fetchTask + submitAction into one class — doesn't match InvokeHandler SPI (one name per handler)
**Rationale:** InvokeHandler.name() returns a single string. Each handler is a single InvokeHandler with one invoke name. Matches the SPI contract directly. No shared state needed between fetchTask and submitAction.
**Trade-offs:** Four classes instead of three. Minimal — each is small.
**Sources:** InvokeHandler.java, InvokeService.java (dispatches by name)
**Exploration:** quick
**Status:** captured

## D3: Canned data — hard-coded fixed responses

**Choice:** Each handler returns a fixed response regardless of input
**Alternatives:**
- Input-sensitive responses (filter by query, match URL patterns) — more realistic but adds complexity unnecessary for contract verification
**Rationale:** Matches the established ref implementation pattern (bank-ref, email-ref, chat-ref). Purpose is demonstrating the handler contract and ComposeAttachment builder usage, not simulating real search/unfurl behaviour.
**Trade-offs:** Not representative of real-world conditional logic. Acceptable — that's what integration tests with real handlers are for.
**Sources:** chat-ref/RefChatPlatform.java, bank-ref pattern, email-ref pattern
**Exploration:** quick
**Status:** captured
