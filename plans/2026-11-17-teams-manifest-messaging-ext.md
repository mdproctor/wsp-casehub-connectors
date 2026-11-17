# Teams Bot Manifest and Messaging Extensions Setup Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #257 — chat-teams: Bot manifest and app registration for messaging extensions
**Issue group:** #257

**Goal:** Create a Teams app manifest template and standalone setup guide so consumers can deploy messaging extensions to a real Teams tenant using the existing InvokeHandler SPI from #236.

**Architecture:** Two documentation artifacts — a JSON manifest template in `chat-teams/src/main/resources/` and a Markdown setup guide in `docs/guides/`. No code changes.

**Tech Stack:** Teams App Manifest schema 1.13, Bot Framework protocol, Quarkus config properties

## Global Constraints

- No code changes — docs and template only
- Manifest schema version >= 1.13 (required for all messaging extension features)
- Raw JSON for card content (consistent with existing `CardUpdate(String cardJson)` pattern)
- Use `_comment` fields for inline JSON documentation (JSON has no comment syntax)

---

## Batch 1: Manifest template and setup guide

### Task 1: Create the Teams app manifest template

**Files:**
- Create: `chat-teams/src/main/resources/manifest-template.json`

**Interfaces:**
- Consumes: Bot Framework manifest schema v1.13
- Produces: Template file referenced by Task 2's setup guide

- [ ] **Step 1: Create the manifest template**

Create `chat-teams/src/main/resources/manifest-template.json` with the following content. This is a complete Teams app manifest with `composeExtensions` entries for all three messaging extension types (search, action, link unfurling):

```json
{
  "$schema": "https://developer.microsoft.com/en-us/json-schemas/teams/v1.13/MicrosoftTeams.schema.json",
  "manifestVersion": "1.13",
  "version": "1.0.0",
  "id": "{{BOT_APP_ID}}",
  "developer": {
    "name": "{{COMPANY_NAME}}",
    "websiteUrl": "https://{{YOUR_DOMAIN}}",
    "privacyUrl": "https://{{YOUR_DOMAIN}}/privacy",
    "termsOfUseUrl": "https://{{YOUR_DOMAIN}}/terms"
  },
  "name": {
    "short": "{{APP_NAME}}",
    "full": "{{APP_FULL_NAME}}"
  },
  "description": {
    "short": "{{SHORT_DESCRIPTION}}",
    "full": "{{FULL_DESCRIPTION}}"
  },
  "icons": {
    "outline": "outline.png",
    "color": "color.png"
  },
  "accentColor": "#FFFFFF",
  "bots": [
    {
      "botId": "{{BOT_APP_ID}}",
      "scopes": ["personal", "team", "groupChat"],
      "supportsFiles": false,
      "isNotificationOnly": false,
      "_comment": "botId must match the Azure Bot registration app ID and the id field above"
    }
  ],
  "composeExtensions": [
    {
      "botId": "{{BOT_APP_ID}}",
      "commands": [
        {
          "_comment": "Search command — dispatched as composeExtension/query. The commandId arrives in InvokeRequest.value().getString('commandId')",
          "id": "searchItems",
          "type": "query",
          "title": "Search items",
          "description": "Search for items by keyword",
          "initialRun": true,
          "parameters": [
            {
              "name": "query",
              "title": "Search query",
              "description": "Enter search terms",
              "inputType": "text"
            }
          ]
        },
        {
          "_comment": "Action command — dispatched as composeExtension/fetchTask (to show form) then composeExtension/submitAction (on submit). Set fetchTask to true to show an Adaptive Card form before submission",
          "id": "createItem",
          "type": "action",
          "title": "Create item",
          "description": "Create a new item via a form",
          "fetchTask": true,
          "parameters": [
            {
              "name": "title",
              "title": "Title",
              "description": "Item title",
              "inputType": "text"
            }
          ]
        }
      ],
      "messageHandlers": [
        {
          "_comment": "Link unfurling — dispatched as composeExtension/queryLink when a user pastes a URL matching these domains",
          "type": "link",
          "value": {
            "domains": [
              "{{YOUR_DOMAIN}}"
            ]
          }
        }
      ]
    }
  ],
  "permissions": ["identity", "messageTeamMembers"],
  "validDomains": [
    "{{YOUR_DOMAIN}}"
  ]
}
```

- [ ] **Step 2: Verify the template is valid JSON**

Run: `python3 -c 'import json; json.load(open("chat-teams/src/main/resources/manifest-template.json")); print("VALID")'`
Expected: `VALID`

- [ ] **Step 3: Commit**

```bash
git add chat-teams/src/main/resources/manifest-template.json
git commit -m "docs(#257): add Teams app manifest template for messaging extensions"
```

### Task 2: Write the standalone setup guide

**Files:**
- Create: `docs/guides/teams-messaging-extensions-setup.md`

**Interfaces:**
- Consumes: Manifest template from Task 1, config properties from `TeamsBeans.java`, InvokeHandler SPI from #236
- Produces: Standalone documentation

- [ ] **Step 1: Write the setup guide**

Create `docs/guides/teams-messaging-extensions-setup.md` with the following content:

```markdown
# Teams Messaging Extensions Setup Guide

Deploy messaging extensions (search, action, link unfurling) to a real
Teams tenant using the casehub-connectors InvokeHandler SPI.

## Prerequisites

Complete these steps in the Azure Portal and Teams Developer Portal
before configuring your application:

- [ ] **Azure Bot registration** — create an Azure Bot resource or Bot
  Channel Registration with a messaging endpoint pointing to your
  application's webhook URL (`https://<host>/webhooks/teams`).
  [Create an Azure Bot](https://learn.microsoft.com/en-us/azure/bot-service/bot-service-quickstart-registration)

- [ ] **Teams channel enabled** — enable the Microsoft Teams channel on
  the bot registration.
  [Connect a bot to Teams](https://learn.microsoft.com/en-us/azure/bot-service/channel-connect-teams)

- [ ] **App ID and password** — note the bot's app ID (also called
  Microsoft App ID) and generate a client secret. The app ID goes into
  Quarkus config; the secret is resolved via `ServiceConnectionProvider`.
  [Manage bot credentials](https://learn.microsoft.com/en-us/azure/bot-service/bot-service-manage-overview)

- [ ] **Teams app package** — create a ZIP containing your manifest,
  plus two icon files (color.png 192x192, outline.png 32x32). Upload
  via Teams Admin Center or sideload for development.
  [Upload a custom app](https://learn.microsoft.com/en-us/microsoftteams/platform/concepts/deploy-and-publish/apps-upload)

## Manifest Template

A ready-to-customise manifest template is provided at:

    chat-teams/src/main/resources/manifest-template.json

Copy this file and replace all `{{PLACEHOLDER}}` values with your
application's details. The template includes:

- A **search command** (`searchItems`) for `composeExtension/query`
- An **action command** (`createItem`) for `composeExtension/fetchTask`
  and `composeExtension/submitAction`
- A **link unfurling handler** via `messageHandlers` for
  `composeExtension/queryLink`

Remove any sections you don't need. Add additional commands by
duplicating entries in the `commands` array — each command needs a
unique `id`.

### Manifest schema version

The template uses schema version **1.13**, the minimum required for all
messaging extension features. Use 1.13 or higher.

## InvokeHandler Wiring

Messaging extension activities arrive as Bot Framework `invoke`
activities. The `TeamsInboundConnector` routes them through
`InvokeService`, which dispatches to `InvokeHandler` CDI beans by
matching `InvokeHandler.name()` to the invoke activity name.

### Activity name mapping

| Manifest element | Invoke activity name | InvokeHandler.name() |
|---|---|---|
| Search command | `composeExtension/query` | `"composeExtension/query"` |
| Search result selected | `composeExtension/selectItem` | `"composeExtension/selectItem"` |
| Action command (fetch form) | `composeExtension/fetchTask` | `"composeExtension/fetchTask"` |
| Action command (submit) | `composeExtension/submitAction` | `"composeExtension/submitAction"` |
| Link unfurling | `composeExtension/queryLink` | `"composeExtension/queryLink"` |
| Settings URL | `composeExtension/querySettingUrl` | `"composeExtension/querySettingUrl"` |
| Settings saved | `composeExtension/setting` | `"composeExtension/setting"` |

### Implementing a handler

Create a CDI bean implementing `InvokeHandler`:

```java
import io.casehub.connectors.chat.teams.InvokeHandler;
import io.casehub.connectors.chat.teams.InvokeRequest;
import io.casehub.connectors.chat.teams.InvokeResponse;
import jakarta.enterprise.context.ApplicationScoped;

@ApplicationScoped
public class MySearchHandler implements InvokeHandler {

    @Override
    public String name() {
        return "composeExtension/query";
    }

    @Override
    public InvokeResponse handle(InvokeRequest request) {
        // request.value() contains the query parameters as JsonObject
        String queryText = request.value().getString("queryText", "");

        String attachmentJson = """
            {
              "contentType": "application/vnd.microsoft.card.adaptive",
              "content": { "type": "AdaptiveCard", "body": [{"type": "TextBlock", "text": "Result for: %s"}] },
              "preview": { "contentType": "application/vnd.microsoft.card.thumbnail", "content": {"title": "Result"} }
            }
            """.formatted(queryText);

        return new InvokeResponse.ComposeResult("list", java.util.List.of(attachmentJson));
    }
}
```

### Response variants

| Variant | Use case | Key fields |
|---|---|---|
| `ComposeResult(attachmentLayout, attachmentsJson)` | Search results, link unfurling cards, selectItem, submitAction results | `attachmentLayout`: `"list"` or `"grid"`. Each attachment: JSON with `contentType`, `content`, optional `preview` |
| `ComposeAuth(suggestedActionsJson)` | SSO/OAuth auth flow initiation | JSON with OAuth URL and action configuration |
| `ComposeContinue(title, cardJson)` | Task module forms for fetchTask and submitAction | `title`: dialog title. `cardJson`: Adaptive Card JSON |
| `Accepted()` | Acknowledgment (e.g. for `composeExtension/setting`) | No fields |
| `Message(text)` | Plain text reply | `text`: message string |

## commandId Routing

When a manifest declares **multiple commands of the same type** (e.g.
two search commands), the invoke activity name is the same for both
(`composeExtension/query`). The `commandId` field inside
`InvokeRequest.value()` distinguishes them.

The `commandId` values must match the `id` field of each command in the
manifest's `composeExtensions[].commands[]` array.

### Worked example

Manifest with two search commands:

```json
{
  "commands": [
    { "id": "searchProducts", "type": "query", "title": "Search products", ... },
    { "id": "searchOrders", "type": "query", "title": "Search orders", ... }
  ]
}
```

Handler routing by `commandId`:

```java
@ApplicationScoped
public class MultiSearchHandler implements InvokeHandler {

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

    private InvokeResponse searchProducts(InvokeRequest request) {
        // Product search logic — return ComposeResult with product cards
        return new InvokeResponse.ComposeResult("list", java.util.List.of(/* ... */));
    }

    private InvokeResponse searchOrders(InvokeRequest request) {
        // Order search logic — return ComposeResult with order cards
        return new InvokeResponse.ComposeResult("list", java.util.List.of(/* ... */));
    }
}
```

**Single-command shortcut:** if a handler serves only one command,
`commandId` routing is unnecessary — process the request directly.

## Quarkus Configuration

| Property | Purpose | Default |
|---|---|---|
| `casehub.connectors.teams-bot.app-id` | Bot Framework app ID (from Azure registration) | _(empty)_ |
| `casehub.connectors.teams-bot.dev-mode` | Skip JWT validation in development | `false` |
| `casehub.connectors.teams-bot.service-url-allowlist` | Allowed Bot Framework service URLs | `*.botframework.com,smba.trafficmanager.net` |
| `casehub.connectors.teams-bot.openid-metadata-url` | OpenID metadata URL for JWT validation | `https://login.botframework.com/v1/.well-known/openidconfiguration` |
| `casehub.connectors.teams-bot.issuer` | Expected JWT issuer | `https://api.botframework.com` |

The bot password/client secret is resolved via `ServiceConnectionProvider`,
not a config property. Configure it through your application's credential
management system.

### Minimal application.properties

```properties
casehub.connectors.teams-bot.app-id=${TEAMS_BOT_APP_ID}
```

For development with sideloaded apps, enable dev mode to skip JWT
validation:

```properties
casehub.connectors.teams-bot.dev-mode=true
```

**Warning:** Never enable dev-mode in production — it disables all
authentication on the Teams webhook endpoint.
```

- [ ] **Step 2: Verify the guide renders correctly**

Check that all Markdown tables and code blocks are well-formed:
Run: `python3 -c 'print("OK")' && wc -l docs/guides/teams-messaging-extensions-setup.md`
Expected: file exists with ~180-200 lines

- [ ] **Step 3: Commit**

```bash
git add docs/guides/teams-messaging-extensions-setup.md
git commit -m "docs(#257): add Teams messaging extensions setup guide"
```

## References

- [2026-11-17-teams-manifest-messaging-ext-design.md] — design spec this plan implements
- `chat-teams/src/main/java/io/casehub/connectors/chat/teams/InvokeHandler.java` — SPI interface
- `chat-teams/src/main/java/io/casehub/connectors/chat/teams/InvokeResponse.java` — sealed response variants
- `chat-teams/src/main/java/io/casehub/connectors/chat/teams/InvokeRequest.java` — request record
- `chat-teams/src/main/java/io/casehub/connectors/chat/teams/TeamsBeans.java` — config properties
- `chat-teams/src/main/java/io/casehub/connectors/chat/teams/BotFrameworkAuth.java` — JWT validation
- casehubio/connectors#236 — messaging extension SPI implementation
- casehubio/connectors#257 — this issue
