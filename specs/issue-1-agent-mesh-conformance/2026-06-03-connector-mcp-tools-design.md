# Connector MCP Tools — Design Spec

**Issue:** casehubio/connectors#1
**Date:** 2026-06-03
**Status:** approved (rev 2 — post code review)

---

## Problem

casehub-connectors is a CDI library — accessible only inside Quarkus apps. The barrier
for external LLM developers is not "cannot import the JAR" but "must write CDI code inside
a Quarkus app to use it." The shift this module makes: any LLM agent can send notifications
by connecting to a running MCP endpoint, with no Quarkus or CDI knowledge required. The
deployment owner runs the Quarkus app; the agent author just calls the tools.

Separately, connectors#1 requires that MCP-initiated deliveries that interact with the
Qhorus agent mesh follow normative patterns — specifically recorded as `EVENT` (no deontic
footprint) on the observe channel, not as obligation-carrying acts.

---

## Solution

A new `mcp` submodule exposes five per-connector MCP tools. Any Quarkus app adds
`casehub-connectors-mcp` to get an MCP surface that LLM agents can call directly.

A `ConnectorMeshBridge` SPI in `core` provides the integration point with Qhorus: when
Qhorus is on the classpath, a real implementation activates and each MCP-initiated
delivery is logged as an `EVENT` on the active observe channel — transparently, without
developer effort. When no case context is active (e.g. a standalone MCP call outside a
case session), the Qhorus implementation behaves as a no-op for that call. EVENT recording
only applies when an MCP call is made within an established case session.

---

## Module Structure

```
core/
  ConnectorMeshBridge          ← new SPI
  NoOpConnectorMeshBridge      ← @DefaultBean, does nothing
  (existing delivery SPIs unchanged)

mcp/                           ← new submodule → artifact: casehub-connectors-mcp
  SlackMcpTool
  TeamsMcpTool
  TwilioSmsMcpTool
  WhatsAppMcpTool
  EmailMcpTool
```

`mcp` depends on `casehub-connectors-core` + `quarkus-mcp-server`. It is optional —
apps that do not want MCP exposure do not add it. No cost when absent.

`ConnectorMeshBridge` lives in `core` (not `mcp`) so that `qhorus/connector-backend`
can implement it by depending only on `casehub-connectors-core`, which it already does.
No new cross-repo dependency needed for the Qhorus bridge.

Five separate tool classes are used rather than a shared base or factory. The per-class
approach gives each tool its own `@Tool(description="...")` and `@ToolArg` metadata with
distinct, connector-specific descriptions — the machine-readable contract that determines
whether LLMs use the tools correctly. A shared implementation skeleton would require
complex annotation-driven registration workarounds; the duplication is justified.

---

## `ConnectorMeshBridge` SPI

```java
// in casehub-connectors-core
public interface ConnectorMeshBridge {
    void notifyDelivered(String connectorId, String destination, String content);
}
```

| Parameter | Value | Source |
|-----------|-------|--------|
| `connectorId` | Connector type constant — `SlackConnector.ID`, `TwilioSmsConnector.ID`, etc. | Caller must use the existing ID constants, never a raw string literal |
| `destination` | Delivery target: webhook URL, E.164 number, or email address | Passed through from the MCP tool parameter |
| `content` | Message body, truncated to 500 chars before passing | Qhorus implementation uses this for the EVENT payload; truncation limits PII exposure |

**SPI contract:**
- Must return quickly. No blocking network I/O on the calling thread. Async/fire-and-forget internally.
- Must tolerate absent case context without throwing. When context cannot be resolved, the implementation silently no-ops for that call.
- Must not throw under any circumstance. Exceptions propagate to the MCP tool caller and surface as errors to the LLM.

`NoOpConnectorMeshBridge` is `@DefaultBean @ApplicationScoped` in `core`. The Qhorus
implementation (tracked in a separate qhorus issue) will be `@ApplicationScoped` without
`@DefaultBean`, displacing the no-op by classpath presence.

**Mesh alignment rule:** the Qhorus implementation must post `EVENT` (not any
obligation-carrying type) to the observe channel. `EVENT` is the only Qhorus message type
with no deontic footprint — correct for telemetry logging of a delivery that has already
occurred.

---

## MCP Tools

Five tools, one per connector. Uniform execution pattern:

```
validate inputs
→ ConnectorService.send()
→ ConnectorMeshBridge.notifyDelivered()   (content truncated to 500 chars)
→ return result string
```

| Tool | Parameters | Routes to |
|------|-----------|-----------|
| `send_slack` | `webhookUrl`, `title`, `body` | `SlackConnector` (`SlackConnector.ID`) |
| `send_teams` | `webhookUrl`, `title`, `body` | `TeamsConnector` (`TeamsConnector.ID`) |
| `send_sms` | `to` (E.164), `body` | `TwilioSmsConnector` (`TwilioSmsConnector.ID`) |
| `send_whatsapp` | `to` (E.164), `body`, `templateName` (optional) | `WhatsAppConnector` (`WhatsAppConnector.ID`) |
| `send_email` | `to`, `subject`, `body` | `EmailConnector` (`EmailConnector.ID`) |

`send_whatsapp` carries an optional `templateName` parameter because WhatsApp Business
API requires pre-approved templates for outbound messages to non-opted-in recipients.
This is the only `ConnectorMessage.attributes` value that is load-bearing enough to
warrant a first-class MCP parameter. All other attributes are out of scope — a stated
limitation, not an oversight.

**Return values:** plain string. `"Dispatched to <destination>"` on success (dispatched,
not confirmed — `ConnectorService.send()` is void and logs failures internally; the MCP
tool cannot distinguish a delivered message from a silently-failed one). `"Failed: <reason>"`
when the connector is not registered or inputs are invalid. Tools never throw.

**Connector credentials** (`send_sms`, `send_whatsapp`, `send_email`) come from Quarkus
config — no credential parameters in the MCP tool signatures.

**Webhook-based tools** (`send_slack`, `send_teams`): the webhook URL is an LLM-controlled
parameter. See Security Model below.

### Tool Descriptions (load-bearing for LLM consumption)

Each tool must carry `@Tool(description="...")` and per-parameter `@ToolArg(description="...")`
annotations. Required content per tool:

| Tool | `@Tool` description must cover | Parameter notes |
|------|-------------------------------|-----------------|
| `send_slack` | Posts a message to a Slack channel via incoming webhook | `webhookUrl`: full Slack incoming webhook URL; `title`: card header (optional); `body`: message text |
| `send_teams` | Posts an adaptive card to a Teams channel via incoming webhook | `webhookUrl`: full Teams webhook URL; `title`: card title; `body`: message text |
| `send_sms` | Sends an SMS via Twilio. Requires Twilio credentials configured on the server | `to`: E.164 format required (e.g. `+447700900000`); `body`: max 1600 chars, longer messages are split |
| `send_whatsapp` | Sends a WhatsApp message via Meta Cloud API. Requires WhatsApp Business credentials on the server | `to`: E.164 format; `body`: message text; `templateName`: required for non-opted-in recipients |
| `send_email` | Sends an email via the configured SMTP server | `to`: recipient email address; `subject`: email subject; `body`: plain-text body |

Exact wording is implementation's choice; the above are the minimum required concepts.

---

## Security Model

`send_slack` and `send_teams` accept LLM-controlled webhook URLs. This is an intentional
design choice for external-facing usability — restricting to config-only URLs would limit
each deployment to one Slack channel, eliminating the use case. The implication is that
a compromised or prompt-injected LLM agent could POST to an attacker-controlled URL.

**Required mitigations (deployment owner's responsibility):**
- The MCP endpoint must be authenticated. `casehub-connectors-mcp` does not implement
  authentication — it is delegated to Quarkus security (OIDC, API key, etc.). An
  unauthenticated endpoint is a public SMS/email/Slack sender.
- Optional hardening: configure a URL allowlist via Quarkus config to restrict permitted
  webhook domains. Not implemented in this module; guidance in the deployment docs.

The SMS/WhatsApp/email tools follow a different trust model: the LLM controls the
recipient, not the sending infrastructure. Deployment owners control credentials and
can restrict usage via authentication scopes.

---

## Testing Strategy

**MCP tools (`mcp` module):**
- `@QuarkusTest` with `quarkus-mcp-server-test` for integration tests — verifies tool
  registration, parameter binding, and return values via MCP test client.
- One happy-path test per tool verifying `ConnectorService.send()` is called with correct
  parameters and `"Dispatched to <destination>"` is returned.
- One error test per tool verifying unregistered connector or missing config returns
  `"Failed: <reason>"`.
- `ConnectorMeshBridge` is `@InjectMock` in MCP tests to verify `notifyDelivered()` is
  called with the correct connectorId constant, destination, and truncated content.

**`ConnectorMeshBridge` SPI (`core` module):**
- Unit test: `NoOpConnectorMeshBridge.notifyDelivered()` does not throw under any input.
- CDI wiring test: when `NoOpConnectorMeshBridge` is the only impl, it is injected as the
  active bridge (verifies `@DefaultBean` displacement pattern).

---

## Qhorus Bridge (Deferred)

The Qhorus implementation of `ConnectorMeshBridge` is out of scope for this issue. A
separate issue will be filed against casehubio/qhorus for `qhorus/connector-backend` to
add a `@ApplicationScoped ConnectorMeshBridge` that:

1. Uses `@RequestScoped` CDI to resolve case context from the active MCP request. MCP
   calls arrive as HTTP requests; a case-scoped session token (populated by a JAX-RS
   filter or auth mechanism) makes the active observe channel resolvable. When no case
   context is present, the implementation silently no-ops — consistent with the contract
   defined in this spec.
2. Posts an `EVENT` message with content truncated to match what `notifyDelivered` receives.
3. Activates by classpath presence — no configuration required.

The qhorus issue must define the case context population mechanism (option: auth token
carries case ID; option: explicit session channel registration). The `ConnectorMeshBridge`
SPI signature does not change regardless of which mechanism is chosen.

Until that issue lands, `NoOpConnectorMeshBridge` is the only active implementation.

---

## Deployment

`casehub-connectors-mcp` does not configure the MCP server transport or authentication.
These are standard Quarkus concerns:
- Transport (SSE vs stdio): configured via `quarkus-mcp-server` properties.
- Authentication: configured via Quarkus security extensions (OIDC, API key, etc.).
- Port and path: standard Quarkus HTTP config.

The module provides the tools; the app provides the server.

---

## Deployment Scenarios

**Standalone (no Qhorus):** add `casehub-connectors-mcp` + connector config. Five MCP
tools available. No mesh awareness. Suitable for any LLM framework.

**With Qhorus (within a case session):** add `casehub-connectors-mcp` + `casehub-qhorus`
(including `connector-backend` with bridge impl). Qhorus bridge activates automatically.
MCP-initiated deliveries within an active case session are recorded as `EVENT` on the
observe channel. No code change required.

---

## Out of Scope

- Inbound MCP tools — the inbound path already handles receipt via CDI event bus and
  `connector-backend` channel bridge. Note: an LLM agent using `send_sms` or
  `send_whatsapp` has no MCP path to read replies — replies arrive via the CDI inbound
  path and must be handled separately. This is a known gap for conversational connectors.
- Generic `send_message(connectorId, ...)` — per-connector tools are self-describing
- Full `ConnectorMessage.attributes` pass-through — only `templateName` (WhatsApp) exposed
- Credential management via MCP parameters
- Retry, scheduling, templating
- URL allowlisting implementation (deployment concern, documented in Security Model)
