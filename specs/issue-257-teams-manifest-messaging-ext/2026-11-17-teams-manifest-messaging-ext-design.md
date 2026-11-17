# Teams Bot Manifest and App Registration for Messaging Extensions

**Issue:** casehubio/connectors#257
**Builds on:** casehubio/connectors#236 (messaging extension SPI)
**Scale:** S | **Complexity:** Low

## Overview

The InvokeHandler SPI and InvokeResponse variants for messaging extensions are complete (#236). Deploying messaging extensions to a real Teams tenant requires a correctly configured Teams app manifest and Azure bot registration. This issue delivers the documentation and template artifacts that bridge the gap between the SPI and a working deployment.

No code changes. Two deliverables: a manifest template file and a standalone setup guide.

## Deliverables

### 1. Manifest template — `chat-teams/src/main/resources/manifest-template.json`

A Teams app manifest (schema version 1.13+) with a `composeExtensions` section declaring all three messaging extension types:

- **Search command** — `composeExtension/query` with parameters
- **Action command** — `composeExtension/fetchTask` / `composeExtension/submitAction`
- **Link unfurling** — `messageHandlers` with URL domain patterns

The template includes:

- `$schema` pointing to the Teams manifest schema
- `manifestVersion` set to `1.13` (minimum for all messaging extension features)
- `id` and `version` placeholders for the consumer's app
- `bots` section with the bot ID placeholder
- `composeExtensions` array with one entry containing:
  - A search command with `parameters` array (title, description, inputType)
  - An action command with `fetchTask: true`
  - `messageHandlers` array for link unfurling domains
- `validDomains` placeholder

Each section has inline JSON comments (via `_comment` fields, since JSON doesn't support comments) explaining what to customise.

### 2. Setup guide — `docs/guides/teams-messaging-extensions-setup.md`

Standalone documentation with these sections:

#### Prerequisites checklist

Bullet list of Azure/Teams setup steps, each with a link to the relevant Microsoft doc:

- Azure Bot registration (or Azure Bot resource)
- Teams channel enabled on the bot
- Messaging endpoint URL configured (`https://<host>/webhooks/teams`)
- App ID and password configured in Quarkus properties
- Teams app package uploaded to the Teams Admin Center or sideloaded

#### Manifest structure walkthrough

Walks through the template file section by section:

- `bots` — bot ID must match the Azure Bot registration app ID
- `composeExtensions` — the `botId` here must also match
- `commands` — each command entry maps to a type of messaging extension
- `messageHandlers` — URL patterns for link unfurling
- `validDomains` — domains the app may load in task modules

#### InvokeHandler wiring

Documents the mapping from manifest to SPI:

| Manifest element | Invoke activity name | InvokeHandler.name() |
|---|---|---|
| Search command | `composeExtension/query` | `"composeExtension/query"` |
| Search result selected | `composeExtension/selectItem` | `"composeExtension/selectItem"` |
| Action command (fetch form) | `composeExtension/fetchTask` | `"composeExtension/fetchTask"` |
| Action command (submit) | `composeExtension/submitAction` | `"composeExtension/submitAction"` |
| Link unfurling | `composeExtension/queryLink` | `"composeExtension/queryLink"` |
| Settings URL | `composeExtension/querySettingUrl` | `"composeExtension/querySettingUrl"` |
| Settings saved | `composeExtension/setting` | `"composeExtension/setting"` |

Each `InvokeHandler` CDI bean returns its `name()` matching the invoke activity name. The `InvokeService` dispatches by exact name match.

#### commandId routing

The key non-obvious section. When a manifest declares multiple commands of the same type (e.g. two search commands), the invoke activity name is the same for both (`composeExtension/query`). The `commandId` distinguishes them — it arrives in `InvokeRequest.value()` as a JSON field:

```java
@ApplicationScoped
public class ProductSearchHandler implements InvokeHandler {

    @Override
    public String name() {
        return "composeExtension/query";
    }

    @Override
    public InvokeResponse handle(InvokeRequest request) {
        String commandId = request.value().getString("commandId", "");
        return switch (commandId) {
            case "searchProducts" -> searchProducts(request);
            case "searchOrders"   -> searchOrders(request);
            default -> new InvokeResponse.Accepted();
        };
    }
}
```

The `commandId` values must match the `id` field of each command in the manifest's `composeExtensions[].commands[]` array.

**Single-command shortcut:** If a handler serves only one command, `commandId` routing is unnecessary — the handler can ignore it and process the request directly.

#### Quarkus configuration

Documents the relevant config properties from `TeamsBeans.java`:

| Property | Purpose | Default |
|---|---|---|
| `casehub.connectors.teams-bot.app-id` | Bot Framework app ID (from Azure registration) | (empty) |
| `casehub.connectors.teams-bot.dev-mode` | Skip JWT validation in dev | `false` |
| `casehub.connectors.teams-bot.service-url-allowlist` | Allowed Bot Framework service URLs | `*.botframework.com,smba.trafficmanager.net` |
| `casehub.connectors.teams-bot.openid-metadata-url` | OpenID metadata for JWT validation | Bot Framework default |
| `casehub.connectors.teams-bot.issuer` | Expected JWT issuer | `https://api.botframework.com` |

The bot password/secret is resolved via `ServiceConnectionProvider` — not a config property.

## What Does NOT Change

- No code modifications — all SPI, serialization, and dispatch code is complete from #236
- Consumer guide (`docs/guides/consumer-guide.md`) — not modified; the standalone doc is self-contained
- No new modules or dependencies

## Files Created

| File | Purpose |
|---|---|
| `chat-teams/src/main/resources/manifest-template.json` | Copyable Teams app manifest template |
| `docs/guides/teams-messaging-extensions-setup.md` | Standalone setup and configuration guide |

## References

- `chat-teams/src/main/java/io/casehub/connectors/chat/teams/InvokeHandler.java` — SPI interface
- `chat-teams/src/main/java/io/casehub/connectors/chat/teams/InvokeResponse.java` — sealed response variants
- `chat-teams/src/main/java/io/casehub/connectors/chat/teams/InvokeResponseSerializer.java` — serialization
- `chat-teams/src/main/java/io/casehub/connectors/chat/teams/InvokeService.java` — dispatch by name
- `chat-teams/src/main/java/io/casehub/connectors/chat/teams/InvokeRequest.java` — request record (carries commandId in value)
- `chat-teams/src/main/java/io/casehub/connectors/chat/teams/TeamsBeans.java` — config properties
- casehubio/connectors#236 — messaging extension SPI implementation
- casehubio/connectors#220 — Teams inbound via Bot Framework (parent)
- `specs/issue-236-teams-messaging-extensions/2026-11-17-teams-messaging-extensions-design.md` — #236 design spec
