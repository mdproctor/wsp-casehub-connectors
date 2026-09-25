## D1: userId handling on capability accessors

**Choice:** Ignore userId — return shared pre-loaded data for all users
**Alternatives:**
- User-partitioned data — each userId gets isolated accounts/transactions. More realistic for multi-user scenarios but adds setup friction and contradicts "zero configuration" goal.
**Rationale:** Matches chat-ref pattern (no user partitioning). Developers wanting user isolation use simulation stubs via `Simulation.forTest()`. Ref is for "does my code work against the SPI" testing, not multi-tenancy testing.
**Trade-offs:** Cannot test user-isolation logic with bank-ref alone — requires simulation stubs or TrueLayer sandbox.
**Sources:** chat-ref/RefChatPlatform.java (no user concept), calendar-ref/RefCalendarPlatform.java (no user concept), issue #109 acceptance criteria ("zero configuration")
**Exploration:** quick
**Status:** captured

## D2: Payment lifecycle simulation strategy

**Choice:** Auto-advance on poll — each `paymentStatus()` call advances to the next state
**Alternatives:**
- Time-based advancement — payment state changes based on elapsed time since creation. More realistic but racy in tests, harder to assert deterministically.
**Rationale:** Deterministic and testable. Developers drive the lifecycle step by step: `initiatePayment()` → AUTHORIZATION_REQUIRED, first `paymentStatus()` → EXECUTED, second → SETTLED, stays at SETTLED thereafter. No timing sensitivity.
**Trade-offs:** Not realistic for demo/integration scenarios where time-based progression matters — simulation stubs or real providers serve that use case.
**Sources:** bank-spi/PaymentStatus.java (enum values), issue #109 acceptance criteria ("simulates payment lifecycle")
**Exploration:** quick
**Status:** captured
