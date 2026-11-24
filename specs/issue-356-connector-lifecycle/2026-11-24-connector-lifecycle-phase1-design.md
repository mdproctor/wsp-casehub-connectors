# ConnectorLifecycle SPI — Phase 1 Design

## Problem

The connectors library has 15+ provider modules, each wiring credentials and CDI beans through bespoke Java code. Three separate credential mechanisms are in use:

- `StaticCredentialStore` — JSON blob keyed by `(actorId, provider, tenancyId)`. Used by Numbrrs, Twilio, Shopify, Amazon, Amadeus.
- `@ConfigProperty` — MicroProfile Config properties from env vars or `application.properties`. Used by Telegram, Signal, IRC.
- `ServiceConnectionProvider` — OAuth token management with scope tracking and refresh. Used by Google (4 connectors), Microsoft (3+), GitHub, TrueLayer.

Credential wiring follows two distinct patterns, neither shared:

- **Explicit CDI producer** (Numbrrs): `NumbrrsMobileBeans` manually resolves credentials from `StaticCredentialStore`, parses the JSON, constructs the client and platform in `@Produces` methods.
- **Annotated class self-wiring** (Telegram, Signal): The platform, connector, and client classes themselves are `@ApplicationScoped` with `@ConfigProperty` constructor injection. The `*Beans.java` classes (`ChatTelegramBeans`, `ChatSignalBeans`) exist only to produce the translator via `new *InboundTranslator()` — they perform no credential wiring.

No shared abstraction connects them.

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
2. **CredentialResolver** — `resolve(providerId)`, look up the manifest field name directly as the map key in the returned property map. `CredentialPropertyKeys` constants are not used — the manifest field names (`api-key`, `bot-token`, `number`, etc.) serve as the canonical keys.
3. **MicroProfile Config** — `casehub.<providerId>.<field>` property (covers env vars, `application.properties`, Vault). Existing config property names that deviate from this convention (e.g. `casehub.telegram.bot.token` instead of `casehub.telegram.bot-token`) are standardized during migration to match the `casehub.<id>.<field>` pattern.

For OAuth providers (Phase 3), `ServiceConnectionProvider.getAccessToken()` is an additional resolution path.

**Multi-tenancy:** Phase 1 resolves credentials at system level only — `StaticCredentialStore.find("system", providerId, "default")` hardcodes actor `"system"` and tenancy `"default"`. This is intentional: Phase 1 targets single-tenant deployments where connector credentials are shared across the system. Tenancy-aware resolution (per-tenant Numbrrs API keys, per-tenant Telegram bots) is a Phase 2+ extension. The facade interface is designed with this migration in mind — an overloaded `resolve(String providerId, String field, String tenancyId)` can be added without breaking existing callers.

The `source()` method enables diagnostics — the MCP `connector_status` tool can report not just whether a credential is configured but where it came from.

### Platform issue

This facade is a platform issue, not a connectors issue. The interface lives in `platform-api` alongside the existing credential infrastructure (`StaticCredentialStore`, `CredentialResolver`, `CredentialPropertyKeys`). The CDI implementation lives in `platform-core` — `platform-api` is pure Java with no CDI. The connectors lifecycle manager depends on the interface but does not implement it.

The facade belongs in `platform-api` because it unifies types that are already in `platform-api`. Adding the facade creates no new dependency for any repo — the dependency is already universal. While connectors is the first consumer, the facade bridges existing platform-scoped credential infrastructure; placing it in `connectors-api` would force future credential consumers to depend on a connectors module.

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
        default: <string>         # optional — fallback value when credential is unresolved

  produces:
    - class: <string>             # fully qualified class name
      scope: <string>             # CDI scope: "application" (default), "dependent"
      type: <string>              # bean type: "platform", "inbound-connector", "translator", "client"
      constructor-args:           # ordered list matching constructor parameter order
        - field: <string>         # resolves credential field via ProviderCredentialFacade (references auth.fields entry)
        - bean: <string>          # injects another produced bean by fully qualified class name

  setup-playbook: <string>        # optional — path to setup playbook YAML (relative to classpath)

  health:
    endpoint: <string>            # optional — health check URL template with {field} substitution
    interval: <string>            # optional — health check interval (default: "60s")
```

**Schema extensibility:** The `auth` section will gain additional top-level keys in Phase 3 for OAuth support (`token-url`, `auth-url`, `scopes`, `redirect-url`). These are OAuth protocol endpoints, not credential fields, so they do not fit in `auth.fields`. The Jackson deserializer uses `FAIL_ON_UNKNOWN_PROPERTIES = false`, ensuring Phase 1 manifests remain valid when Phase 3 keys are added. The `family` field is already reserved for Phase 3.

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
            @JsonProperty("sensitive") boolean sensitive,
            @JsonProperty("default") String defaultValue) {

        public FieldDeclaration {
            if (name == null || name.isBlank())
                throw new IllegalArgumentException("field name required");
        }

        // defaults: required=true, sensitive=true, no default value
        public FieldDeclaration(String name, String description) {
            this(name, true, description, true, null);
        }
    }

    public record BeanDeclaration(
            @JsonProperty("class") String className,
            String scope,
            String type,
            @JsonProperty("constructor-args") List<ConstructorArg> constructorArgs) {}

    public record ConstructorArg(
            String field,
            String bean) {
        public ConstructorArg {
            if ((field == null) == (bean == null))
                throw new IllegalArgumentException("exactly one of field or bean required");
        }
    }

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
      constructor-args:
        - field: api-key
        - field: numbers
    - class: io.casehub.connectors.mobile.numbrrs.NumbrrsInboundConnector
      type: inbound-connector
      constructor-args:
        - field: signing-secret
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
        default: "https://api.telegram.org"
        description: "Telegram API base URL"

  produces:
    - class: io.casehub.connectors.telegram.bot.TelegramBotClient
      type: client
      constructor-args:
        - field: api-url
    - class: io.casehub.connectors.chat.telegram.TelegramChatPlatform
      type: platform
      constructor-args:
        - bean: io.casehub.connectors.telegram.bot.TelegramBotClient
        - field: bot-token
        - field: webhook-secret
    - class: io.casehub.connectors.chat.telegram.TelegramInboundConnector
      type: inbound-connector
      constructor-args:
        - field: webhook-secret
        - bean: io.casehub.connectors.chat.telegram.TelegramChatPlatform
    - class: io.casehub.connectors.chat.telegram.TelegramInboundTranslator
      type: translator

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
    - class: io.casehub.connectors.signal.cli.SignalClient
      type: client
      constructor-args:
        - field: api-url
    - class: io.casehub.connectors.chat.signal.SignalChatPlatform
      type: platform
      constructor-args:
        - bean: io.casehub.connectors.signal.cli.SignalClient
        - field: api-url
        - field: number
    - class: io.casehub.connectors.chat.signal.SignalInboundConnector
      type: inbound-connector
      constructor-args:
        - bean: io.casehub.connectors.signal.cli.SignalClient
        - field: api-url
        - field: number
    - class: io.casehub.connectors.chat.signal.SignalInboundTranslator
      type: translator

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
   └─ Validate: required fields present in schema, referenced classes exist,
       constructor-args field references match auth.fields entries,
       constructor-args bean references match other produces entries

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

The synthetic bean creation function is driven entirely by the `constructor-args` declarations in the manifest. Each entry maps to a constructor parameter in order:

- `field: <name>` — resolves the credential field via `ProviderCredentialFacade.resolve(connectorId, name)`. If the facade returns empty/null and the field's `FieldDeclaration` has a `default` value, the default is used instead. The resolved (or default) value is adapted to the constructor parameter type using Jandex type introspection at build time:
  - `String` parameter — passed directly (empty string if unresolved and no default)
  - `Optional<String>` parameter — wrapped via `Optional.ofNullable()`
  - Additional types (`int`, `boolean`, `Duration`, `List<String>`) are added in Phase 2+ as needed. Phase 1 supports `String` and `Optional<String>` only.
- `bean: <class>` — injects the referenced bean from the CDI container via `SyntheticCreationalContext.getInjectedReference()`.
- Entries with no `constructor-args` — no-arg constructor (e.g. translators).

The build extension contains no connector-specific construction code. The manifest is the single source of truth for how each bean is assembled. Inter-bean dependencies (e.g. `TelegramChatPlatform` depending on `TelegramBotClient`) are declared via `bean:` references and resolved by Quarkus's CDI container — no explicit ordering or topological sort is needed.

**Degraded mode:** When required credentials are missing, the build extension still registers the bean and passes empty credential values. The platform classes handle degradation internally via their existing `initDegraded()` pattern — both `TelegramChatPlatform` and `SignalChatPlatform` check for blank credentials in their `@PostConstruct init()` and swap all capability interfaces to no-op implementations (e.g. `EmptyDiscovery`, `NoOpReactions`). There are no separate `NoOp*Platform` classes. For `NumbrrsMobilePlatform`, which currently throws `IllegalStateException` on missing credentials, the migration adds an `initDegraded()` path for consistency. The `ConnectorRegistry` records connectors with missing required credentials as `UNCONFIGURED` rather than `READY`.

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

**Relationship with existing tools:** `ConnectorOperationsImpl` in the `graphql` module already provides `connectorStatus` (lists registered connectors and active capabilities) and `connectorsReport` (per-platform capability summary with scope filter) under `@McpDomain("connectors")`. These existing tools report runtime capability state — which capabilities are active on registered platforms. The new lifecycle tools report a different dimension: manifest discovery, credential configuration status, health, and setup playbook state. The two sets coexist with clear boundaries:

- **Existing `connectorStatus` / `connectorsReport`**: runtime capabilities of registered platform beans (chat, calendar, bank, email, mobile). Scope: all connector types, including those not yet manifest-driven.
- **New `connector_discover` / `connector_status`**: lifecycle state of manifest-declared connectors. Scope: only connectors with a `connector-manifest.yaml`.

As more connectors migrate to manifests in Phase 2+, the lifecycle tools will subsume the existing tools. Naming uses the new `snake_case` convention under the `connectors/lifecycle` domain — no collision with the existing `camelCase` names in the `connectors` domain.

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

### Template variable resolution

Playbook templates use `{variable}` substitution. Variables are resolved in order:

1. **Auth fields** — the connector manifest's `auth.fields` entries, resolved via `ProviderCredentialFacade` (including defaults)
2. **System config** — MicroProfile Config properties under the `casehub.playbook.*` namespace (e.g. `casehub.playbook.webhook-base-url`)
3. **Step captures** — output from prior steps (e.g. `{verification.code}` from a `wait-for-sms` capture)

Deployment-level variables like `webhook-base-url` are system config, not auth fields — they describe the deployment, not the connector. Configure via `casehub.playbook.webhook-base-url` in `application.properties` or environment variable `CASEHUB_PLAYBOOK_WEBHOOK_BASE_URL`.

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
| `health-check` | HTTP GET, fail if non-2xx | playbooks |
| `http-post` | HTTP POST with JSON body | playbooks |
| `connector-status` | Check a connector's status via the lifecycle manager | connectors lifecycle |
| `wait-for-sms` | Poll for SMS via `MobilePlatformService` SPI until a matching message arrives | playbooks |

**Step type contracts:**

- **`health-check`**: Input: `url` (string, supports `{field}` substitution). Output: captured as the step name (e.g. `bot-info`). Behaviour: HTTP GET, retries 3 times with 2s backoff, fails the playbook if all retries return non-2xx.
- **`http-post`**: Input: `url` (string), `body` (JSON object). Output: response body captured as the step name. Behaviour: HTTP POST with `Content-Type: application/json`, fails on non-2xx.
- **`connector-status`**: Input: `connector-id` (string), `require` (status enum). Behaviour: queries `ConnectorRegistry`, fails if the connector's status doesn't match `require`.
- **`wait-for-sms`**: Input: `provider` (connector id), `to` (phone number), `pattern` (regex with capture group), `timeout` (duration). Output: `code` capture group from first matching message. Behaviour: polls `MobilePlatformService.platform(provider).sms().listMessages()` every 5s until a message matching `pattern` arrives or `timeout` expires. The step uses the `MobilePlatformService` SPI — the playbooks module has no direct dependency on `mobile-numbrrs`.

## Migration Plan

### connectors-api

- Add `ConnectorManifest` record and supporting types
- Add `ConnectorManifestLoader`
- Add `@ConnectorBean` qualifier annotation

### New: connector-lifecycle module

A new module (or a `lifecycle` package within `connectors-api`) containing:

- `ConnectorLifecycleBuildExtension` — the Quarkus build extension
- `ConnectorRegistry` — runtime registry of discovered connectors and their lifecycle status (READY, UNCONFIGURED, ERROR). This is distinct from `ConnectorService`, which routes outbound messages to `Connector` implementations, and from `ChatPlatformService`/`MobilePlatformService`, which provide platform-type-specific SPI registries. `ConnectorRegistry` knows about ALL manifest-declared beans across all types and tracks their credential/health lifecycle — a dimension the existing services don't cover.
- `ConnectorLifecycleOperations` — `@McpDomain("connectors/lifecycle")` MCP tool implementations

### mobile-numbrrs

- Add `src/main/resources/META-INF/connector-manifest.yaml`
- Delete `NumbrrsMobileBeans.java`
- `NumbrrsMobilePlatform` constructor changes from `(NumbrrsClient client, List<String> numbers)` to `(String apiKey, String numbers)` — creates `NumbrrsClient` internally and parses the comma-separated `numbers` string. This matches the manifest's `constructor-args: [{field: api-key}, {field: numbers}]`.
- `NumbrrsInboundConnector` constructor unchanged — already takes `(String signingSecret)`
- `NumbrrsClient` constructor unchanged (already takes `apiKey` and `baseUrl`)
- Add `initDegraded()` path: when `apiKey` is blank, construct a no-op platform instead of throwing `IllegalStateException`

### chat-telegram

- Add `src/main/resources/META-INF/connector-manifest.yaml`
- Delete `ChatTelegramBeans.java`
- Remove `@ApplicationScoped` from `TelegramChatPlatform`, `TelegramInboundConnector`, `TelegramBotClient`
- Remove `@ConfigProperty` injection — credentials arrive via constructor parameters from the synthetic bean
- Standardize config property names: `casehub.telegram.bot.token` → `casehub.telegram.bot-token`, `casehub.telegram.webhook.secret-token` → `casehub.telegram.webhook-secret`
- Retain `@PostConstruct`-equivalent init logic in the constructor or a factory method
- Fix `TelegramInboundConnector.handle()`: replace `String.equals()` webhook secret comparison with `MessageDigest.isEqual()` for constant-time verification (per `PP-20260529-b7765c`)
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

Not in Phase 1 scope — see "What Is NOT in Phase 1" below.

## Testing Strategy

### Unit tests (connectors-api / connector-lifecycle)

- `ConnectorManifest` YAML parsing: valid manifests, missing required fields, malformed YAML, unknown keys (forward-compatibility)
- `ConstructorArg` validation: exactly-one-of field/bean constraint
- `ConnectorManifestLoader` merge cascade: override priority ordering, field-level replacement by connector id
- `ProviderCredentialFacade` resolution cascade ordering: StaticCredentialStore → CredentialResolver → MicroProfile Config, with source tracking

### Integration tests (connector-lifecycle)

- Synthetic bean registration: Quarkus augmentation test verifying that beans declared in a test manifest are injectable
- Constructor-args wiring: test manifest with field + bean references, verify correct credential values and bean injection
- Degraded mode: test with missing required credentials, verify platform enters degraded state and ConnectorRegistry records UNCONFIGURED
- ConnectorRegistry population: verify all manifest-declared connectors appear with correct status

### MCP tool tests

- `connector_discover`: returns all manifest-declared connectors with correct status and field information
- `connector_status`: returns credential source information, health check results
- `connector_setup`: triggers playbook execution for connectors with setup-playbook
- `connector_health`: runs health checks, returns per-connector results

### Regression tests (migrated connectors)

- Numbrrs: webhook delivery still works (signing verification, SMS/voice event handling)
- Telegram: webhook delivery still works, degraded mode activates on missing bot-token, webhook secret comparison uses constant-time `MessageDigest.isEqual()`
- Signal: WebSocket connection still works, degraded mode activates on missing api-url or number

### Playbook step type tests

- `health-check`: success on 200, retry on 503, fail after retries exhausted
- `http-post`: success with response capture, fail on non-2xx
- `connector-status`: pass when connector matches required status, fail otherwise
- `wait-for-sms`: message matching with regex capture, timeout behaviour

## What Is NOT in Phase 1

| Concern | Phase | Reason |
|---------|-------|--------|
| Family definitions (Google, Microsoft) | Phase 3 | OAuth families require scope merging and incremental consent — different complexity tier |
| Migration of all API-key connectors | Phase 2 | Twilio, Shopify, Amazon, Amadeus follow the pattern proved by Phase 1 |
| Browser automation for OAuth | Phase 3 | Playwright-driven consent flows depend on the OAuth family model |
| Mosaic web UI | Future | MCP tools are the primary API; UI consumes them later |
| Credential rotation/revocation via manifest | Phase 2+ | `StaticCredentialStore` already handles this; the facade exposes it |
| connectors-spring manifest-driven beans | Phase 2 | Spring `@Configuration` reads the same manifests and registers beans via `@Bean` methods; the manifest schema is shared but the bean registration mechanism differs; Quarkus proves the model first |

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
