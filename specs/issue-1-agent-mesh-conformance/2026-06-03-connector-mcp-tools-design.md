# Connector MCP Tools — Design Spec

**Issue:** casehubio/connectors#1
**Date:** 2026-06-03
**Status:** approved

---

## Problem

casehub-connectors is a CDI library — accessible only inside Quarkus apps. External LLM
developers who want battle-tested Slack/SMS/email/WhatsApp delivery cannot use it without
adopting the full CaseHub stack. This creates a chicken-and-egg adoption barrier.

Separately, connectors#1 requires that any future connector interacting with the Qhorus
agent mesh follows normative patterns — specifically that delivered messages are recorded
as `EVENT` (no deontic footprint) on the observe channel, not as obligation-carrying acts.

---

## Solution

A new `mcp` submodule exposes five per-connector MCP tools. Any Quarkus app adds
`casehub-connectors-mcp` to get an MCP surface that LLM agents can call directly.

A `ConnectorMeshBridge` SPI in `core` provides the integration point with Qhorus: when
Qhorus is on the classpath, a real implementation activates and each delivery is logged
as an `EVENT` on the active observe channel — transparently, without developer effort.

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

---

## `ConnectorMeshBridge` SPI

```java
// in casehub-connectors-core
public interface ConnectorMeshBridge {
    void notifyDelivered(String connectorId, String destination, String content);
}
```

| Parameter | Semantics |
|-----------|-----------|
| `connectorId` | Connector type: `"slack"`, `"teams"`, `"twilio-sms"`, `"whatsapp"`, `"email"` |
| `destination` | Delivery target: webhook URL, E.164 number, or email address |
| `content` | Message body — needed by the Qhorus implementation to construct the EVENT payload |

`NoOpConnectorMeshBridge` is `@DefaultBean @ApplicationScoped` in `core`. The Qhorus
implementation (tracked in a separate qhorus issue) will be `@ApplicationScoped` without
`@DefaultBean`, displacing the no-op by classpath presence.

**Mesh alignment rule:** the Qhorus implementation must post `EVENT` (not any
obligation-carrying type) to the observe channel. `EVENT` is the only Qhorus message type
with no deontic footprint — correct for telemetry logging of a delivery that has already
occurred.

---

## MCP Tools

Five tools, one per connector. Uniform pattern: validate inputs → `ConnectorService.send()`
→ `ConnectorMeshBridge.notifyDelivered()` → return result string.

| Tool | Parameters | Routes to |
|------|-----------|-----------|
| `send_slack` | `webhookUrl`, `title`, `body` | `SlackConnector` (`"slack"`) |
| `send_teams` | `webhookUrl`, `title`, `body` | `TeamsConnector` (`"teams"`) |
| `send_sms` | `to` (E.164), `body` | `TwilioSmsConnector` (`"twilio-sms"`) |
| `send_whatsapp` | `to` (E.164), `body` | `WhatsAppConnector` (`"whatsapp"`) |
| `send_email` | `to`, `subject`, `body` | `EmailConnector` (`"email"`) |

**Return values:** plain string. `"Delivered to <destination>"` on success. `"Failed: <reason>"` on error. Tools never throw — errors surface as return strings so the calling LLM can handle them gracefully.

**Connector credentials** (`send_sms`, `send_whatsapp`, `send_email`) come from Quarkus
config as normal — no credential parameters in the MCP tool signatures.

**Webhook-based tools** (`send_slack`, `send_teams`): the webhook URL IS the
`ConnectorMessage.destination` — no config lookup needed.

---

## Qhorus Bridge (Deferred)

The Qhorus implementation of `ConnectorMeshBridge` is out of scope for this issue. A
separate issue will be filed against casehubio/qhorus for `qhorus/connector-backend` to
add a `@ApplicationScoped ConnectorMeshBridge` that:

1. Resolves the active Qhorus observe channel for the current context (the qhorus issue
   must define what "current context" means when no case is active — e.g. a global
   connector observe channel, or a no-op when context is absent)
2. Posts an `EVENT` message: `"Connector delivery: {connectorId} → {destination}: {content}"`
3. Activates by classpath presence — no configuration required

Until that issue lands, `NoOpConnectorMeshBridge` is the only active implementation and
all deployments behave as standalone delivery only.

---

## Deployment Scenarios

**Standalone (no Qhorus):** add `casehub-connectors-mcp` + connector config. Five MCP
tools available. No mesh awareness. Suitable for any LLM framework.

**With Qhorus:** add `casehub-connectors-mcp` + `casehub-qhorus` (including
`connector-backend`). Qhorus bridge activates automatically. Every delivery is recorded
as `EVENT` on the observe channel. No code change required.

---

## Out of Scope

- Inbound MCP tools (receive messages via MCP) — the inbound path already has a full
  CDI event bus and `connector-backend` channel bridge; MCP is not needed there
- Generic `send_message(connectorId, ...)` tool — per-connector tools are self-describing
  and more useful to LLMs
- Credential management via MCP — credentials stay in Quarkus config
- Retry, scheduling, templating — connectors is delivery infrastructure only
