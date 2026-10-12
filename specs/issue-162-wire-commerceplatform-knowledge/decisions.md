# Decisions — Issue #162: Wire CommercePlatform to MCP domain

## D1: Follow ConnectorBankApi pattern for McpDomain wiring

**Choice:** Create `ConnectorCommerceApi` following the exact `ConnectorBankApi` pattern — `@McpDomain("connectors/commerce")`, inject `CommercePlatformService` + `SecurityIdentity`, capability-check before delegating to SPI.
**Alternatives:**
- Custom MCP tools — manual tool definitions without McpDomain auto-generation. More flexible but inconsistent with all other platform APIs.
- Generic commerce operations through `ConnectorOperationsImpl` — keeps one class but violates separation between operational status and platform-specific CRUD.
**Rationale:** Every other platform SPI (bank, calendar, chat, contacts, document, email, project) follows this exact pattern. Consistency reduces cognitive load and ensures auto-generated REST + MCP from one source.
**Trade-offs:** None significant — this is the established pattern.
**Sources:** `ConnectorBankApi.java`, `ConnectorContactsApi.java`, `ConnectorOperationsImpl.java`
**Exploration:** quick
**Status:** captured

## D2: @RequiresConfirmation handles consent transparently

**Choice:** The `@RequiresConfirmation` annotation on `Checkout.checkout()` handles consent at the SPI interceptor level. The `ConnectorCommerceApi` delegates directly without explicit consent-checking logic.
**Alternatives:**
- Explicit `WorkerAuthorizationPolicy` check in API layer — adds consent logic in the wrong layer, duplicates SPI-level enforcement.
- Separate consent issue — defers integration, but consent is already built.
**Rationale:** Platform interceptors handle `@RequiresConfirmation` transparently for all SPI calls. Adding explicit checks in the API layer would duplicate enforcement and couple the API to consent implementation details.
**Trade-offs:** If the interceptor chain isn't active (e.g., direct SPI usage without CDI), consent is skipped — but that's the SPI's responsibility, not the API layer's.
**Sources:** `CommercePlatform.Checkout`, platform#509 (WorkerAuthorizationPolicy consent tiers)
**Exploration:** quick
**Status:** captured
