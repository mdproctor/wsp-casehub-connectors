# ConnectorLifecycle SPI — Phase 1 Design

## Problem

The connectors library has 15+ provider modules, each wiring credentials and CDI beans through bespoke Java code. Three separate credential mechanisms are in use:

- `StaticCredentialStore` — JSON blob keyed by `(actorId, provider, tenancyId)`. Used by Numbrrs, Twilio, Shopify, Amazon, Amadeus.
- `@ConfigProperty` — MicroProfile Config properties from env vars or `application.properties`. Used by Telegram, Signal, IRC.
- `ServiceConnectionProvider` — OAuth token management with scope tracking and refresh. Used by Google (4 connectors), Microsoft (3+), GitHub, TrueLayer.

Each connector has its own `*Beans.java` CDI producer class. Some use explicit `@Produces` methods that manually construct everything (Numbrrs pattern). Others use direct `@ApplicationScoped` on the platform class with `@Inject` constructor injection (Telegram/Signal pattern). No shared abstraction connects them.

The consequences:

1. **Agents cannot discover or configure connectors.** There is no API to ask "what connectors are available?", "which ones are configured?", or "set up this connector for me." An agent that needs to use Numbrrs SMS must know the specific credential key structure and store it manually.
2. **Every new connector requires bespoke Java.** Adding a provider means writing a `*Beans.java` class, choosing one of three credential patterns, and implementing init logic by hand.
3. **No health or readiness reporting.** Some connectors degrade silently when credentials are missing (Telegram, Signal). Others throw at startup (Numbrrs). Neither approach tells an agent or user what is configured and what is not.

## Phase 1 Scope

Foundation infrastructure plus three first-mover migrations that prove the model on three distinct auth patterns:

| Connector | Auth type | Inbound | Init steps |
|-----------|-----------|---------|------------|
| Numbrrs | API key (StaticCredentialStore) | Webhook push | None |
| Telegram | Bot token (@ConfigProperty) | Webhook push | Optional webhook registration |
| Signal | Phone number (@ConfigProperty) | WebSocket pull | Health check + phone verification via SMS |

**Deliverables:**

1. `connector-manifest.yaml` schema and `ConnectorManifest` record (connectors)
2. `ConnectorManifestLoader` with merge cascade (connectors)
3. Quarkus build extension producing synthetic CDI beans from manifests (connectors)
4. `ProviderCredentialFacade` unified credential resolution (platform — separate issue)
5. MCP tools: `connector_discover`, `connector_setup`, `connector_status`, `connector_health` (connectors)
6. Setup playbooks for Telegram webhook registration and Signal phone verification (connectors)
7. Migration of Numbrrs, Telegram, Signal to manifest-driven wiring (connectors)

## Platform Dependency: ProviderCredentialFacade

The lifecycle manager needs a single interface to resolve credentials regardless of storage backend. Platform already owns all three credential mechanisms (`StaticCredentialStore`, `CredentialResolver`, `ServiceConnectionProvider`) in `platform-api`. The missing piece is a facade that bridges them.

### Interface

```java
package io.casehub.platform.api.credentials;

public interface ProviderCredentialFacade {

    Optional<String> resolve(String providerId, String field);

    Map<String, String> resolveAll(String providerId);

    boolean isConfigured(String providerId, List<String> requiredFields);

    CredentialSource source(String providerId, String field);

    enum CredentialSource { CREDENTIAL_STORE, CONFIG, ENVIRONMENT, SERVICE_CONNECTION, NONE }
}
```

### Resolution cascade

For a given `(providerId, field)` pair, the facade checks in order:

1. **StaticCredentialStore** — `find("system", providerId, "default")`, parse the JSON `credential` string, extract the named field.
2. **CredentialResolver** — `resolve(providerId)`, look up field in the returned property map using `CredentialPropertyKeys`.
3. **MicroProfile Config** — `casehub.<providerId>.<field>` property (covers env vars, `application.properties`, Vault).

For OAuth providers (Phase 3), `ServiceConnectionProvider.getAccessToken()` is an additional resolution path.

The `source()` method enables diagnostics — the MCP `connector_status` tool can report not just whether a credential is configured but where it came from.

### Platform issue

This facade is a platform issue, not a connectors issue. It lives in `platform-api` alongside the existing credential infrastructure. The connectors lifecycle manager depends on it but does not implement it.

**Suggested issue:** "ProviderCredentialFacade — unified credential resolution across StaticCredentialStore, CredentialResolver, and MicroProfile Config" in `casehubio/platform`.

## Manifest Schema

Each connector module includes a `connector-manifest.yaml` in `src/main/resources/META-INF/`. The file follows platform YAML conventions established by `voice-config.yaml` and `agent-config.yaml`: kebab-case keys, record-based Jackson deserialization via `@JsonProperty`, parsing with `YamlMappers.create()`.

### Schema

```yaml
connector:
  id: <string>                    # unique connector identifier (e.g. "numbrrs", "telegram", "signal")
  platform: <string>              # SPI platform type (e.g. "mobile", "chat", "calendar")
  description: <string>           # human-readable description

  auth:
    type: <string>                # auth pattern: "api-key", "bot-token", "phone-verify", "oauth"
    family: <string>              # optional family reference (e.g. "google", "microsoft") — Phase 3
    fields:
      - name: <string>            # credential field name (e.g. "api-key", "bot-token")
        required: <boolean>       # default: true
        description: <string>     # human-readable description for agents/UI
        sensitive: <boolean>      # default: true — masked in status output

  produces:
    - class: <string>             # fully qualified class name
      scope: <string>             # CDI scope: "application" (default), "dependent"
      type: <string>              # bean type: "platform", "inbound-connector", "translator", "client"

  setup-playbook: <string>        # optional — path to setup playbook YAML (relative to classpath)

  health:
    endpoint: <string>            # optional — health check URL template with {field} substitution
    interval: <string>            # optional — health check interval (default: "60s")
```

### Java records

```java
package io.casehub.connectors.lifecycle;

public record ConnectorManifest(
        ConnectorDeclaration connector) {

    public record ConnectorDeclaration(
            String id,
            String platform,
            String description,
            AuthDeclaration auth,
            List<BeanDeclaration> produces,
            @JsonProperty("setup-playbook") String setupPlaybook,
            HealthDeclaration health) {}

    public record AuthDeclaration(
            String type,
            String family,
            List<FieldDeclaration> fields) {}

    public record FieldDeclaration(
            String name,
            @JsonProperty("required") boolean required,
            String description,
            @JsonProperty("sensitive") boolean sensitive) {

        public FieldDeclaration {
            if (name == null || name.isBlank())
                throw new IllegalArgumentException("field name required");
        }

        // defaults: required=true, sensitive=true
        public FieldDeclaration(String name, String description) {
            this(name, true, description, true);
        }
    }

    public record BeanDeclaration(
            @JsonProperty("class") String className,
            String scope,
            String type) {}

    public record HealthDeclaration(
            String endpoint,
            String interval) {}
}
```

### First-mover manifests

**Numbrrs** (`mobile-numbrrs/src/main/resources/META-INF/connector-manifest.yaml`):

```yaml
connector:
  id: numbrrs
  platform: mobile
  description: "Numbrrs — UK mobile SIM numbers for inbound SMS and voice"

  auth:
    type: api-key
    fields:
      - name: api-key
        description: "Numbrrs REST API key"
      - name: signing-secret
        required: false
        description: "HMAC-SHA256 webhook signing secret"
      - name: numbers
        required: false
        sensitive: false
        description: "Comma-separated phone numbers"

  produces:
    - class: io.casehub.connectors.mobile.numbrrs.NumbrrsMobilePlatform
      type: platform
    - class: io.casehub.connectors.mobile.numbrrs.NumbrrsInboundConnector
      type: inbound-connector
```

**Telegram** (`chat-telegram/src/main/resources/META-INF/connector-manifest.yaml`):

```yaml
connector:
  id: telegram
  platform: chat
  description: "Telegram Bot API — messaging, threading, discovery, reactions, presence, commands"

  auth:
    type: bot-token
    fields:
      - name: bot-token
        description: "Telegram Bot API token from @BotFather"
      - name: webhook-secret
        required: false
        description: "Secret token for webhook request verification"
      - name: api-url
        required: false
        sensitive: false
        description: "Telegram API base URL (default: https://api.telegram.org)"

  produces:
    - class: io.casehub.connectors.chat.telegram.TelegramChatPlatform
      type: platform
    - class: io.casehub.connectors.chat.telegram.TelegramInboundConnector
      type: inbound-connector
    - class: io.casehub.connectors.chat.telegram.TelegramInboundTranslator
      type: translator
    - class: io.casehub.connectors.telegram.bot.TelegramBotClient
      type: client

  setup-playbook: playbooks/telegram-webhook-setup.yaml

  health:
    endpoint: "{api-url}/bot{bot-token}/getMe"
```

**Signal** (`chat-signal/src/main/resources/META-INF/connector-manifest.yaml`):

```yaml
connector:
  id: signal
  platform: chat
  description: "Signal via signal-cli-rest-api — messaging, discovery, reactions, groups"

  auth:
    type: phone-verify
    fields:
      - name: api-url
        sensitive: false
        description: "signal-cli-rest-api base URL (e.g. http://localhost:8080)"
      - name: number
        sensitive: false
        description: "Signal phone number in E.164 format (e.g. +447700100001)"

  produces:
    - class: io.casehub.connectors.chat.signal.SignalChatPlatform
      type: platform
    - class: io.casehub.connectors.chat.signal.SignalInboundConnector
      type: inbound-connector
    - class: io.casehub.connectors.chat.signal.SignalInboundTranslator
      type: translator
    - class: io.casehub.connectors.signal.cli.SignalClient
      type: client

  setup-playbook: playbooks/signal-registration.yaml

  health:
    endpoint: "{api-url}/v1/health"
```

## ConnectorManifestLoader

Discovers and loads manifests using the same merge cascade pattern as `VoiceManifestLoader` and `AgentManifestLoader`.

### Discovery

At build time (Quarkus extension) and at runtime (for status reporting), the loader scans:

1. `META-INF/connector-manifest.yaml` on the classpath — each connector module contributes its own
2. `/etc/casehub/connector-manifest.yaml` — system-level overrides
3. `~/.casehub/connector-manifest.yaml` — user-level overrides
4. Project directory `connector-manifest.yaml` — project-level overrides
5. Profile variants (e.g. `connector-manifest-dev.yaml`) — environment-specific overrides

Overrides can selectively replace fields for a given connector ID without duplicating the entire manifest. The merge uses connector `id` as the key — a later source with the same `id` replaces fields from earlier sources.

### Java

```java
package io.casehub.connectors.lifecycle;

public class ConnectorManifestLoader {

    public static List<ConnectorManifest> discover() {
        // scan META-INF/connector-manifest.yaml across classpath
        // apply merge cascade
        // validate required fields
        // return ordered list
    }
}
```

## Lifecycle Build Extension

A Quarkus build extension that replaces the bespoke `*Beans.java` CDI producers with a uniform manifest-driven mechanism.

### Build-time flow

```
1. ConnectorManifestDiscoveryBuildStep
   └─ Scan classpath for META-INF/connector-manifest.yaml
   └─ Parse each into ConnectorManifest
   └─ Validate: required fields present in schema, referenced classes exist

2. ConnectorSyntheticBeanBuildStep (for each manifest)
   └─ For each entry in produces:
       └─ Register SyntheticBeanBuildItem
           ├─ Bean class: the declared class
           ├─ Scope: @ApplicationScoped (default)
           ├─ Creation function:
           │   1. Inject ProviderCredentialFacade
           │   2. Resolve auth fields by connector id
           │   3. Construct the bean (see Construction below)
           └─ Qualifiers: @ConnectorBean(id = "<connector-id>")

3. ConnectorStatusBuildStep
   └─ Register a ConnectorRegistry singleton
   └─ Populated with all discovered manifests + their configuration state
```

### Runtime bean construction

The synthetic bean creation function follows a convention based on the `type` field in the manifest:

| Type | Construction pattern |
|------|---------------------|
| `client` | Constructor takes resolved credential fields as parameters |
| `platform` | Constructor takes the client bean + additional config |
| `inbound-connector` | Constructor takes signing/verification credentials |
| `translator` | No-arg constructor (pure data mapper) |

For Phase 1, the construction logic is connector-specific — each connector's classes expose constructors that accept the resolved credential values. The build extension knows how to call them based on the manifest's `produces` declarations.

**Degraded mode:** When required credentials are missing, the build extension still registers the bean but wraps it in a degraded proxy. For platform beans, this uses the existing `NoOp*Platform` pattern. The `ConnectorRegistry` records the connector as `UNCONFIGURED` rather than `READY`.

### @ConnectorBean qualifier

```java
@Qualifier
@Retention(RUNTIME)
@Target({TYPE, METHOD, FIELD, PARAMETER})
public @interface ConnectorBean {
    String id();
}
```

Enables injection by connector ID when multiple connectors produce the same SPI type (e.g. multiple `ChatPlatform` implementations).

## MCP Tools

Exposed via `@McpDomain("connectors/lifecycle")`, these tools give agents and users a uniform interface to discover, configure, and monitor connectors.

### connector_discover

Lists all connectors found via manifest discovery, with their auth requirements and configuration status.

```json
{
  "connectors": [
    {
      "id": "numbrrs",
      "platform": "mobile",
      "description": "Numbrrs — UK mobile SIM numbers for inbound SMS and voice",
      "status": "READY",
      "auth-type": "api-key",
      "required-fields": ["api-key"],
      "optional-fields": ["signing-secret", "numbers"],
      "setup-playbook": null
    },
    {
      "id": "signal",
      "platform": "chat",
      "description": "Signal via signal-cli-rest-api",
      "status": "UNCONFIGURED",
      "auth-type": "phone-verify",
      "required-fields": ["api-url", "number"],
      "missing-fields": ["api-url", "number"],
      "setup-playbook": "playbooks/signal-registration.yaml"
    }
  ]
}
```

### connector_status

Detailed status for a specific connector or all connectors. Includes credential resolution source, health check results, and setup playbook status.

```json
{
  "id": "telegram",
  "status": "READY",
  "credentials": {
    "bot-token": { "configured": true, "source": "CREDENTIAL_STORE" },
    "webhook-secret": { "configured": true, "source": "CONFIG" },
    "api-url": { "configured": false, "source": "NONE", "default": "https://api.telegram.org" }
  },
  "health": { "status": "HEALTHY", "checked-at": "2026-11-24T10:00:00Z" },
  "setup-playbook": "playbooks/telegram-webhook-setup.yaml",
  "setup-status": "NOT_RUN"
}
```

### connector_setup

Triggers the setup playbook for a connector. For connectors without a playbook, validates that credentials are configured and runs the health check.

Parameters: `connector-id` (required), `dry-run` (optional, default false).

### connector_health

Runs health checks across all configured connectors. Returns a summary with per-connector status.

## Setup Playbooks

Connectors that need init steps beyond credential wiring ship a YAML playbook. The playbook infrastructure already exists in the `playbooks` module — setup playbooks are standard playbooks that happen to run during connector initialisation rather than as user-triggered workflows.

### Telegram webhook setup

`playbooks/telegram-webhook-setup.yaml`:

```yaml
name: Telegram webhook registration
description: Register the webhook URL with Telegram Bot API
---
- health-check:
    url: "{api-url}/bot{bot-token}/getMe"
  capture: bot-info

- http-post:
    url: "{api-url}/bot{bot-token}/setWebhook"
    body:
      url: "{webhook-base-url}/connectors/telegram-inbound/webhook"
      secret_token: "{webhook-secret}"
      allowed_updates: ["message", "edited_message", "channel_post", "callback_query", "my_chat_member"]
  capture: webhook-result
```

### Signal registration

`playbooks/signal-registration.yaml`:

```yaml
name: Signal phone registration
description: Register a phone number with Signal using SMS verification via Numbrrs
---
- health-check:
    url: "{api-url}/v1/health"

- connector-status:
    connector-id: numbrrs
    require: READY

- http-post:
    url: "{api-url}/v1/register/{number}"
    body:
      use_voice: false
  capture: register-request

- wait-for-sms:
    provider: numbrrs
    to: "{number}"
    pattern: "Your Signal verification code.*?(\\d{3}-\\d{3})"
    timeout: 120s
  capture: verification

- http-post:
    url: "{api-url}/v1/register/{number}/verify/{verification.code}"
  capture: verify-result
```

**New playbook step types required:**

| Step type | Purpose | Module |
|-----------|---------|--------|
| `health-check` | HTTP GET, fail if non-2xx | connectors-api or playbooks |
| `http-post` | HTTP POST with JSON body | connectors-api or playbooks |
| `connector-status` | Check a connector's status via the lifecycle manager | connectors lifecycle |
| `wait-for-sms` | Poll Numbrrs `listMessages()` until a matching SMS arrives | mobile-numbrrs or playbooks |

## Migration Plan

### connectors-api

- Add `ConnectorManifest` record and supporting types
- Add `ConnectorManifestLoader`
- Add `@ConnectorBean` qualifier annotation

### New: connector-lifecycle module

A new module (or a `lifecycle` package within `connectors-api`) containing:

- `ConnectorLifecycleBuildExtension` — the Quarkus build extension
- `ConnectorRegistry` — runtime registry of discovered connectors and their status
- `ConnectorLifecycleOperations` — `@McpDomain("connectors/lifecycle")` MCP tool implementations

### mobile-numbrrs

- Add `src/main/resources/META-INF/connector-manifest.yaml`
- Delete `NumbrrsMobileBeans.java`
- `NumbrrsMobilePlatform` and `NumbrrsInboundConnector` gain constructors that accept resolved credential values directly (no `StaticCredentialStore` injection)
- `NumbrrsClient` constructor unchanged (already takes `apiKey` and `baseUrl`)

### chat-telegram

- Add `src/main/resources/META-INF/connector-manifest.yaml`
- Delete `ChatTelegramBeans.java`
- Remove `@ApplicationScoped` from `TelegramChatPlatform`, `TelegramInboundConnector`, `TelegramBotClient`
- Remove `@ConfigProperty` injection — credentials arrive via constructor parameters from the synthetic bean
- Retain `@PostConstruct`-equivalent init logic in the constructor or a factory method
- Add `playbooks/telegram-webhook-setup.yaml`

### chat-signal

- Add `src/main/resources/META-INF/connector-manifest.yaml`
- Delete `ChatSignalBeans.java`
- Remove `@ApplicationScoped` from `SignalChatPlatform`, `SignalInboundConnector`, `SignalClient`
- Remove `@ConfigProperty` injection — credentials arrive via constructor parameters
- Retain health check logic in constructor
- Add `playbooks/signal-registration.yaml`

### signal-cli

- `SignalClient` becomes a plain class (remove `@ApplicationScoped`, `@ConfigProperty`)
- Constructor takes `apiUrl` as a parameter

### telegram-bot

- `TelegramBotClient` becomes a plain class (remove `@ApplicationScoped`, `@ConfigProperty`)
- Constructor takes `baseUrl` as a parameter

### connectors-spring

- Equivalent mechanism for Spring Boot: a `@Configuration` class that reads manifests and registers beans via `@Bean` methods. The manifest schema is shared; the bean registration mechanism differs.

## What Is NOT in Phase 1

| Concern | Phase | Reason |
|---------|-------|--------|
| Family definitions (Google, Microsoft) | Phase 3 | OAuth families require scope merging and incremental consent — different complexity tier |
| Migration of all API-key connectors | Phase 2 | Twilio, Shopify, Amazon, Amadeus follow the pattern proved by Phase 1 |
| Browser automation for OAuth | Phase 3 | Playwright-driven consent flows depend on the OAuth family model |
| Mosaic web UI | Future | MCP tools are the primary API; UI consumes them later |
| Credential rotation/revocation via manifest | Phase 2+ | `StaticCredentialStore` already handles this; the facade exposes it |

## References

- D1–D12 in `specs/issue-356-connector-lifecycle/decisions.md`
- `NumbrrsMobileBeans.java` — `mobile-numbrrs/src/main/java/.../NumbrrsMobileBeans.java`
- `ChatTelegramBeans.java` — `chat-telegram/src/main/java/.../ChatTelegramBeans.java`
- `TelegramChatPlatform.java` — `chat-telegram/src/main/java/.../TelegramChatPlatform.java`
- `TelegramInboundConnector.java` — `chat-telegram/src/main/java/.../TelegramInboundConnector.java`
- `TelegramBotClient.java` — `telegram-bot/src/main/java/.../TelegramBotClient.java`
- `ChatSignalBeans.java` — `chat-signal/src/main/java/.../ChatSignalBeans.java`
- `SignalChatPlatform.java` — `chat-signal/src/main/java/.../SignalChatPlatform.java`
- `SignalClient.java` — `signal-cli/src/main/java/.../SignalClient.java`
- `StaticCredentialStore.java` — `platform-api/src/main/java/.../authn/StaticCredentialStore.java`
- `CredentialResolver.java` — `platform-api/src/main/java/.../credentials/CredentialResolver.java`
- `ServiceConnectionProvider.java` — `platform-api/src/main/java/.../authn/ServiceConnectionProvider.java`
- `VoiceManifest.java` / `VoiceManifestLoader.java` — `voice-spi/src/main/java/.../voice/config/`
- `AgentManifest.java` / `AgentManifestLoader.java` — `agent-config-core/src/main/java/.../agent/config/`
- Quarkus `SyntheticBeanBuildItem` API
- Existing `@McpDomain("connectors")` pattern in `graphql` module 
