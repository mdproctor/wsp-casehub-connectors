# Design Journal — issue-1-agent-mesh-conformance

### 2026-06-04 · §Module Structure

Added `casehub-connectors-mcp` as a fifth Maven module. Depends on `core` + `email` + `quarkus-mcp-server-core:1.11.1`. The `mcp` module separation keeps `core` dependency-free — it carries no MCP infrastructure, so CDI-only consumers take no quarkiverse dependency. Consuming apps add `quarkus-mcp-server-http` for the transport alongside `casehub-connectors-mcp`.

### 2026-06-04 · §SPI

`ConnectorMeshBridge` SPI added to `core` (not `mcp`) so `qhorus/connector-backend` can implement it by depending only on `casehub-connectors-core`, which it already does — no new cross-repo dependency introduced. `NoOpConnectorMeshBridge @DefaultBean @Unremovable` is the default; Qhorus bridge activates by classpath presence (tracked in qhorus#249).

`@Unremovable` is required because the injection point (`@Inject ConnectorMeshBridge`) lives in the `mcp` module, not in `core`. Without it, ARC eliminates the bean at augmentation time when `core` is used standalone (no `mcp` on classpath). Same pattern as `@DefaultBean` no-ops throughout the platform.

SPI contract: must return quickly (no blocking I/O on calling thread), must tolerate absent case context without throwing (Qhorus bridge no-ops when no case session is active), must never throw. `null` is explicitly permitted for `content` — implementations must treat it as empty.

### 2026-06-04 · §Design Principles

Bridge is called from MCP tools after successful `ConnectorService.send()` — NOT from inside `ConnectorService` itself. Rationale: CDI-only callers (casehub-engine, casehub-work) are already in a CaseHub context with full Qhorus observability; logging their deliveries to the observe channel a second time would be redundant and incorrect. The bridge is an MCP-surface concern only.

Bridge receives sanitized, truncated content (500 chars, all ASCII control chars ≤ 0x1F and DEL 0x7F replaced with space) to prevent log injection at the Qhorus observe-channel layer. Original full body is passed to the underlying connector unchanged.

Tools catch `Exception` broadly (not just `IllegalArgumentException`) and return `"Failed: "` strings — the "tools never throw" contract must hold even for unexpected connector-level failures. Raw stack traces must not reach the LLM caller.

### 2026-06-04 · §Usage

Five MCP tools: `send_slack`, `send_teams`, `send_sms`, `send_whatsapp`, `send_email`. All follow the pattern: validate inputs → `ConnectorService.send()` → `ConnectorMeshBridge.notifyDelivered()` → return `"Dispatched to <destination>"`.

`send_whatsapp` has two optional parameters: `templateName` (routes to WhatsApp template message API instead of free-form text) and `templateLanguage` (BCP-47 code, defaults to `en_US`). These map to `ConnectorMessage.attributes` keys `templateName` and `templateLanguage` consumed by `WhatsAppConnector.buildPayload()`.

`send_sms` and `send_whatsapp` use E.164 phone numbers as destination. `send_slack` and `send_teams` use webhook URLs (the URL IS the credential). `send_email` uses an email address as destination and maps `subject` to `ConnectorMessage.title`.
