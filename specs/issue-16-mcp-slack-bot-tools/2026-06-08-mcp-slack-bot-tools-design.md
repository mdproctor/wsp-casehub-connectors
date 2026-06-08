# MCP Tools for SlackBot — Design Spec

**Issue:** connectors#16  
**Branch:** issue-16-mcp-slack-bot-tools  
**Date:** 2026-06-08

---

## Problem

The `casehub-connectors-mcp` module exposes `send_slack` via incoming webhook URL. Webhook
URLs are per-channel credentials the caller supplies at call time — agents cannot discover
available channels, cannot reply to threads, and must manage URLs as secrets in every call.

`SlackBotClient` (shipped in connectors#2) provides a richer model: a single configured bot
token enables multi-channel posting, thread replies, and channel discovery via the Slack Web
API. The MCP surface has not been updated to reflect this.

---

## Goals

1. Expose `send_slack_bot` — bot-token-based Slack posting that returns the message `ts` for
   thread replies.
2. Expose `list_channels` — generic channel discovery across all connectors that support it.
3. Introduce `ConnectorDiscovery` as a reusable SPI so Discord, Telegram, and the planned
   Quarkus demo chat service can register discoverable targets without a new MCP tool each time.
4. Fix the latent `@Blocking` omission on all existing MCP tools.

---

## Design

### `ConnectorDiscovery` SPI — `casehub-connectors-core`

```java
public interface ConnectorDiscovery {
    String connectorId();
    List<DiscoveredTarget> discover();

    record DiscoveredTarget(String id, String displayName) {}
}
```

- `@ApplicationScoped` CDI beans, discovered automatically alongside `Connector` beans.
- Completely separate from the `Connector` SPI — a connector may implement one, both, or neither.
- `connectorId()` links results to their source connector for `list_channels` output formatting.
- `DiscoveredTarget.id` is what the agent passes to `send_slack_bot`; `displayName` is the
  human-readable label (e.g. `#general`).
- No pagination contract in the SPI — each implementation decides internally.

### `SlackBotClient` changes — `casehub-connectors-slack-bot`

**Implements `ConnectorDiscovery`:**

```java
@ApplicationScoped
public class SlackBotClient implements ConnectorDiscovery {

    @ConfigProperty(name = "casehub.connectors.slack-bot.token", defaultValue = "")
    String botToken;   // package-private, like apiBaseUrl

    @Override
    public String connectorId() { return "slack-bot"; }

    @Override
    public List<DiscoveredTarget> discover() {
        // GET /api/conversations.list?types=public_channel,private_channel&limit=200
        // Returns DiscoveredTarget(id, "#" + name) for each channel
        // Returns empty list if botToken is blank or Slack returns ok=false
    }
}
```

**Configured-token `postMessage` overload:**

```java
public PostResult postMessage(String channelId, String text, String threadTs) {
    return postMessage(botToken, channelId, text, threadTs);
}
```

The existing `postMessage(String token, String channelId, String text, String threadTs)`
remains unchanged — `casehub-qhorus-slack-channel` passes its own token.

**Internal parsing types** (private nested records):

```java
private record ListChannelsResult(boolean ok, List<ChannelSummary> channels, String error) {}
private record ChannelSummary(String id, String name) {}
```

`discover()` calls `HttpHelper.CLIENT.send()` (blocking). The `@Blocking` annotation lives on
the MCP tool method, not on `SlackBotClient` methods.

### `SlackBotMcpTool` — `casehub-connectors-mcp`

Injects `SlackBotClient` directly — does not route through `ConnectorService`. The `Connector`
SPI is void (no return value) and has no threading concept; forcing bot posting through it
would discard the `ts`.

```java
@ApplicationScoped
public class SlackBotMcpTool {

    private final SlackBotClient slackBotClient;
    private final ConnectorMeshBridge meshBridge;

    @Inject
    SlackBotMcpTool(SlackBotClient slackBotClient, ConnectorMeshBridge meshBridge) { ... }

    @Tool(name = "send_slack_bot",
          description = "Posts a message to a Slack channel using a configured bot token. "
                      + "Returns the message timestamp (ts) on success — save it to reply "
                      + "in-thread. Requires casehub.connectors.slack-bot.token on the server. "
                      + "Returns 'Posted to <channel> (ts=<ts>)' on success or "
                      + "'Failed: <reason>' on error.")
    @Blocking
    public String sendSlackBot(
            @ToolArg(description = "Channel ID (e.g. C123ABC) or name (e.g. #general). "
                                 + "Use list_channels to discover available channels.")
            String channel,
            @ToolArg(description = "Message text.")
            String text,
            @ToolArg(description = "Thread timestamp for in-thread replies. Use the ts from "
                                 + "a previous send_slack_bot call. Omit for a new message.",
                     required = false)
            String threadTs) {
        try {
            PostResult result = slackBotClient.postMessage(channel, text, threadTs);
            if (!result.ok()) {
                return "Failed: " + result.error();
            }
            meshBridge.notifyDelivered("slack-bot", channel,
                    McpContentSanitizer.sanitize(text));
            return "Posted to " + channel + " (ts=" + result.ts() + ")";
        } catch (Exception e) {
            LOG.warnf("send_slack_bot failed [%s]: %s", e.getClass().getSimpleName(),
                    e.getMessage());
            return "Failed: " + e.getMessage();
        }
    }
}
```

**Bridge called only on `result.ok()`** — `SlackBotClient` catches HTTP errors internally
and returns `PostResult(ok=false)`. An ok=false result is not a delivered message.

### `ChannelDiscoveryMcpTool` — `casehub-connectors-mcp`

```java
@ApplicationScoped
public class ChannelDiscoveryMcpTool {

    private final Iterable<ConnectorDiscovery> discoveries;

    @Inject
    ChannelDiscoveryMcpTool(@Any Instance<ConnectorDiscovery> discoveries) {
        this.discoveries = discoveries;
    }

    // Package-private — test constructor
    ChannelDiscoveryMcpTool(Iterable<ConnectorDiscovery> discoveries) {
        this.discoveries = discoveries;
    }

    @Tool(name = "list_channels",
          description = "Lists discoverable channels across all configured connectors "
                      + "(e.g. Slack Bot). Use the returned IDs when calling send_slack_bot. "
                      + "Only connectors with a token configured appear in the output.")
    @Blocking
    public String listChannels() {
        StringBuilder sb = new StringBuilder();
        for (ConnectorDiscovery d : discoveries) {
            List<DiscoveredTarget> targets = d.discover();
            if (targets.isEmpty()) continue;
            sb.append(d.connectorId()).append(":\n");
            for (DiscoveredTarget t : targets) {
                sb.append("  ").append(t.displayName())
                  .append(" (").append(t.id()).append(")\n");
            }
        }
        return sb.isEmpty() ? "No channels discovered." : sb.toString().stripTrailing();
    }
}
```

No bridge call — discovery is a read operation.

`Iterable<ConnectorDiscovery>` as the field type accepts both `Instance<ConnectorDiscovery>`
(CDI) and `List<ConnectorDiscovery>` (test constructor) — both implement `Iterable`.

### `send_slack` (webhook) — unchanged

The existing `send_slack` tool remains as-is. It serves a different use case: no bot setup
required, caller supplies the webhook URL per call. The two tools are complementary, not
competing.

### `@Blocking` on existing MCP tools

All five existing tools call `HttpHelper.CLIENT.send()` through `ConnectorService`. None have
`@Blocking`. Per garden GE-20260604-96d82a, blocking calls from a `@Tool` method require
`@Blocking` to avoid stalling the Vert.x I/O thread. One-line fix on each:

- `SlackMcpTool.sendSlack`
- `TeamsMcpTool.sendTeams`
- `TwilioSmsMcpTool.sendSms`
- `WhatsAppMcpTool.sendWhatsApp`
- `EmailMcpTool.sendEmail`

### Module dependency

`mcp/pom.xml` gains:

```xml
<dependency>
  <groupId>io.casehub</groupId>
  <artifactId>casehub-connectors-slack-bot</artifactId>
  <version>0.2-SNAPSHOT</version>
</dependency>
```

### Configuration

| Property | Default | Description |
|---|---|---|
| `casehub.connectors.slack-bot.token` | `""` | Bot token (`xoxb-…`). Blank → `discover()` returns empty list; `postMessage` via configured-token overload sends with blank token (Slack returns `invalid_auth`). |

### Return format

| Scenario | Return |
|---|---|
| `send_slack_bot` success | `"Posted to #general (ts=1638535627.000200)"` |
| `send_slack_bot` Slack error | `"Failed: channel_not_found"` |
| `send_slack_bot` HTTP/network error | `"Failed: <exception message>"` |
| `list_channels` with results | Multi-line formatted string (see above) |
| `list_channels` nothing discovered | `"No channels discovered."` |

---

## Testing

### `SlackBotClientTest` additions

- `discover_returnsChannelList` — WireMock stubs `GET /api/conversations.list`, verifies
  `DiscoveredTarget` list with correct id and `#`-prefixed displayName.
- `discover_blankToken_returnsEmptyList` — sets `botToken = ""`, verifies empty list returned
  without HTTP call.
- `discover_slackReturnsNotOk_returnsEmptyList` — WireMock returns `{"ok":false}`.
- `postMessage_configuredToken_usesStoredToken` — verifies the 3-param overload delegates to
  the 4-param overload with `botToken`.

### `SlackBotMcpToolTest`

Lives in package `io.casehub.connectors.slack.bot` (in `mcp` module test sources) to access
`apiBaseUrl` and `botToken` package-private fields. Uses WireMock and `RecordingBridge`.

- `sendSlackBot_success_returnsPostedWithTs` — stubs Slack to return `ok=true`, verifies
  return includes `ts`.
- `sendSlackBot_slackReturnsNotOk_returnsFailedNoBridgeCall` — stubs `ok=false`; bridge must
  not be called.
- `sendSlackBot_networkError_returnsFailedString`
- `sendSlackBot_withThreadTs_passesThreadTsToClient`
- `sendSlackBot_longText_contentTruncatedTo500InBridge`
- `sendSlackBot_success_bridgeCalledWithConnectorIdChannelAndSanitizedText`

### `ChannelDiscoveryMcpToolTest`

Uses package-private test constructor with `List.of(stub)`. `StubDiscovery` inner class
implementing `ConnectorDiscovery`.

- `listChannels_singleConnector_formatsOutput`
- `listChannels_multipleConnectors_formatsAll`
- `listChannels_emptyDiscover_skipsConnector`
- `listChannels_noConnectors_returnsNoneDiscovered`

### Existing MCP tool tests

No logic changes — verify `@Blocking` addition compiles cleanly (tests continue to pass).

---

## Deferred Concerns

- parent#197 — Update PLATFORM.md capability ownership table
- parent#198 — Sync casehub-connectors.md deep-dive

---

## Platform Coherence

- `send_slack_bot` calls `meshBridge.notifyDelivered()` on `ok=true` → observe channel
  receives delivery event consistent with existing tools.
- `ConnectorDiscovery` SPI has no external SDK types in its signature.
- `discover()` uses `HttpHelper.CLIENT` (shared-http-client protocol).
- All `@Tool` methods that call blocking HTTP annotated `@Blocking` (GE-20260604-96d82a).
- No same-name `@Tool` overloads on any class (GE-20260430-b015f5).
- `@ToolArg(required = false)` for optional `threadTs` (GE-20260414-fa6489).
