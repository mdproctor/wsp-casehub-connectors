# MCP Server Packaging — Design Spec

**Issue:** casehubio/connectors#324 (epic), #323 (implementation)
**Branch:** issue-324-mcp-server-packaging
**Date:** 2026-11-19

## 1. Goal

Package the platform's @McpDomain tools as a standalone MCP server consumable by any MCP client — langchain4j, Claude Desktop, OpenAI agents, Cursor, etc. The server is a single Quarkus app that can be built as a JVM JAR or GraalVM native image, exposing tools from one or more domain projects (connectors, neocortex, etc.) based on classpath composition.

### Design evolution

The epic (#324) originally proposed a native proxy + JVM backend architecture — a native image frontend serving `tools/list` instantly while proxying `tools/call` to a JVM backend. During design review, this was replaced with a simpler single-process model because Quarkus ArC performs all CDI wiring at build time and is fully native-image-safe. The two-process architecture solved a problem that doesn't exist — `quarkus-mcp-server` already compiles to native image. See decisions.md D2, D3, D5, D9 (superseded) for the full reasoning.

## 2. Architecture

### Single-process Quarkus app — reusing existing platform infrastructure

The server reuses the existing platform `mcp` module unchanged. The following components already exist and are production-tested:

- **`GraphQLModelScanner`** (platform `mcp/`) — discovers `@McpDomain` beans via CDI at startup
- **`DynamicToolRegistrar`** (platform `mcp/`) — registers tools via `ToolManager` from `quarkus-mcp-server`
- **`ReflectiveOperationDispatcher`** (platform `mcp/`) — dispatches tool calls to domain beans

No custom MCP protocol handling is needed — `quarkus-mcp-server` handles all protocol concerns today.

```
┌──────────────────────────────────────────────────┐
│  MCP Server (Quarkus app)                        │
│                                                  │
│  ┌──────────────┐  ┌──────────────────────────┐  │
│  │ quarkus-mcp- │  │ GraphQLModelScanner      │  │
│  │ server       │  │ (EXISTING — discovers    │  │
│  │ (EXISTING)   │  │  @McpDomain beans)       │  │
│  │ stdio / HTTP │  └──────────┬───────────────┘  │
│  └──────┬───────┘             │                  │
│         │          ┌──────────▼───────────────┐  │
│         │          │ DynamicToolRegistrar     │  │
│         │          │ + ReflectiveDispatcher   │  │
│         └──────────┤ (EXISTING)              │  │
│                    └──────────┬───────────────┘  │
│                               │                  │
│         ┌─────────────────────┼──────────┐       │
│         ▼                     ▼          ▼       │
│  ┌────────────┐  ┌────────────┐  ┌───────────┐  │
│  │ connectors │  │ neocortex  │  │ other     │  │
│  │ @McpDomain │  │ @McpDomain │  │ @McpDomain│  │
│  │ beans      │  │ beans      │  │ beans     │  │
│  │ (EXISTING) │  │            │  │           │  │
│  └─────┬──────┘  └────────────┘  └───────────┘  │
│        │                                         │
│  ┌─────▼──────┐  ┌────────────┐                  │
│  │ *-ref      │  │ *-google   │  (provider       │
│  │ (default)  │  │ *-microsoft│   activation     │
│  │ (EXISTING) │  │ *-slack    │   via config)    │
│  └────────────┘  └────────────┘                  │
└──────────────────────────────────────────────────┘
```

### Two build outputs

| Mode | Command | Startup | Providers |
|------|---------|---------|-----------|
| JVM | `java -jar casehub-mcp-server.jar` | 2-5s | All — including reflection-heavy Google providers |
| Native | `./casehub-mcp-server` | ~50ms | Native-ready only — ref, Microsoft, others per audit |

The native image audit (committed at `docs/audits/2026-11-19-native-image-readiness.md`) identifies 30 modules as ready, 17 likely ready, 8 needing work, 6 that won't work in native. JVM mode is the primary deployment; native is an optimisation for startup-sensitive use cases.

### Transports

- **stdio** (default): for local MCP clients (Claude Desktop, langchain4j CLI, Cursor). Process reads/writes JSON-RPC on stdin/stdout.
- **Streamable HTTP** (`--transport=http`): for remote/container deployments. Bidirectional communication over standard HTTP (MCP spec 2025-03-26). Supersedes SSE transport.

Both are provided by `quarkus-mcp-server` — no custom protocol handling.

### Tool exposure model

The standalone server exposes all @McpDomain tools individually at startup — flat tool registration, not the platform's `casehub_action` meta-dispatch pattern. External LLM clients need individual tool schemas to reason about when to use each tool. The existing `DynamicToolRegistrar` already supports both modes; the standalone server configuration selects flat mode.

For tool catalog size concerns (90+ tools may overwhelm some LLMs), explicit mode domain filtering (§4) allows consumers to limit the exposed tool set.

## 3. Composability

The server is domain-agnostic. Tool availability is determined by what @McpDomain JARs are on the classpath. `GraphQLModelScanner` auto-discovers all @McpDomain beans — no manifest files, no build-time catalog generation. Whatever CDI beans are present get registered as tools.

| Distribution | Classpath includes | Tool count |
|---|---|---|
| `casehub-mcp-server-connectors` | connectors graphql + all *-ref + deps | ~90 tools |
| `casehub-mcp-server-neocortex` | neocortex @McpDomain + deps | TBD |
| `casehub-mcp-server-combined` | both | ~90 + TBD |

Each distribution is a thin config module — a `pom.xml` declaring dependencies + `application.properties` for defaults.

### Tool name uniqueness

Tool name uniqueness within a single compilation unit is validated at build time by `GraphQLResolverProcessor` (existing APT). Cross-module name uniqueness (e.g. connectors tool colliding with a platform tool in a combined distribution) is validated at runtime by `GraphQLModelScanner` at startup — duplicate tool names cause a startup failure with a clear error.

## 4. Domain filtering (explicit mode)

A single config property controls which domains are exposed:

```properties
# Discover mode (default) — all domains on classpath are exposed
casehub.mcp.mode=discover

# Explicit mode — only listed domains exposed
casehub.mcp.mode=explicit
casehub.mcp.domains=chat,calendar,document,location
```

For the initial release, filtering is applied at the `tools/list` response level — tools from excluded domains are not registered with `ToolManager`. All providers still load (they're CDI beans on the classpath). This is the simplest approach and doesn't require modifying existing domain API classes.

Build-time CDI elimination (via `@IfBuildProperty` or Quarkus build extensions) is a future optimisation for resource-constrained deployments — it would prevent excluded providers from loading at all, saving memory and startup time, but requires adding conditional annotations to existing code.

## 5. Provider activation

Ref implementations are the default for every platform SPI. Real providers activate via config:

```properties
# Calendar: ref → Google
casehub.calendar.provider=google
casehub.calendar.google.credentials.path=/path/to/credentials.json

# Email: ref → Microsoft
casehub.email.provider=microsoft
casehub.email.microsoft.client-id=...
casehub.email.microsoft.client-secret=...

# Chat: ref → Slack (can configure multiple)
casehub.chat.provider=slack
casehub.chat.slack.bot-token=xoxb-...
```

When no provider config is present, ref implementations serve realistic seed data — immediately useful for prototyping and development.

## 6. Platform changes (in `platform` repo)

### 6.1 Add `mcp-server` module (new)

A JAR module (`<packaging>jar</packaging>`) containing:
- Anonymous `CurrentPrincipal` provider (`@DefaultBean`) for standalone use — overridden when the server runs within the full platform
- Domain filtering support — `tools/list` filtering by `casehub.mcp.domains` config
- Default `application.properties` for standalone MCP server operation
- Flat tool registration mode configuration (individual tools, not `casehub_action` dispatch)

This is the only new platform module. No changes to existing `mcp-api`, `mcp-core`, `mcp`, or `mcp-spring` modules.

### 6.2 Relationship to Spring Boot path

The platform already has a Spring Boot MCP path (`mcp-spring/`, `connectors-spring/`). The two paths coexist:
- **Quarkus path** (this spec): primary, full-featured, native image capable
- **Spring Boot path** (existing): for Spring Boot applications that want to embed connectors. `connectors-spring/` auto-configuration is untested as a standalone server but could be extended independently.

This spec does not modify or deprecate the Spring path.

## 7. Connectors changes (in `connectors` repo)

### 7.1 Config module: `mcp-server-connectors` (new module)

Thin packaging module with no Java source code beyond CDI wiring:

**`pom.xml` dependencies (full tree):**

Compile scope:
- `casehub-platform-mcp-server` (new platform module)
- `casehub-platform-mcp` (existing — scanner, registrar, dispatcher)
- `casehub-connectors-graphql` (existing — all 15 @McpDomain domain API classes)
- `casehub-connectors-core` (ConnectorService, InboundConnectorService)
- All 13 SPI modules: `chat-spi`, `calendar-spi`, `bank-spi`, `email-spi`, `document-spi`, `contacts-spi`, `project-spi`, `location-spi`, `commerce-spi`, `travel-spi`, `health-spi`, `voice-spi`, `mobile-spi`
- All 13 ref modules: `chat-ref`, `calendar-ref`, `bank-ref`, `email-ref`, `document-ref`, `contacts-ref`, `project-ref`, `location-ref`, `commerce-ref`, `travel-ref`, `health-ref`, `voice-ref`, `mobile-ref`
- `connectors-api` (shared types: Page, HttpHelper, SyncResult, etc.)
- `notification-bridge` (DeliveryChannelRegistry)

Runtime scope:
- `quarkus-mcp-server-sse` (MCP protocol — transitive via platform `mcp`)

**`application.properties`:**
```properties
casehub.mcp.mode=discover
casehub.mcp.tool-exposure=flat
quarkus.mcp-server.transport=stdio
quarkus.http.host=127.0.0.1
```

**Maven profiles:**
- `jvm` (default): standard Quarkus uber-jar
- `native`: GraalVM native image with resource config for ref seed YAML

### 7.2 `ConnectorOperationsImpl` CDI wiring

Add `@ApplicationScoped` to `ConnectorOperationsImpl`. The class already has constructor injection for all 16 parameters — adding the scope annotation is sufficient for CDI discovery.

### 7.3 Startup test

An integration test that starts the Quarkus app and verifies:
- `tools/list` returns all expected tools
- `tools/call` with a ref provider returns seed data
- Domain filtering excludes the right tools

## 8. Authentication and security

### stdio transport
Inherits process identity. No auth needed — the process runs as the user.

### HTTP transport
- **Bind to `127.0.0.1` by default** — not `0.0.0.0`. Local-only access.
- Binding to a non-loopback address requires explicit config: `quarkus.http.host=0.0.0.0`
- Startup WARNING logged if bound to non-loopback without auth configured
- No built-in auth for the initial release. Users deploying over a network should front with a reverse proxy (nginx, Envoy) for auth.

### `CurrentPrincipal`
Anonymous principal in standalone mode (provided by platform `mcp-server` module).

Future work: API key auth, OAuth2, multi-tenant tool catalogs.

## 9. Implementation sequence

1. **Platform: add `mcp-server` module** — anonymous CurrentPrincipal, domain filtering, flat tool mode config
2. **Connectors: add `@ApplicationScoped` to `ConnectorOperationsImpl`**
3. **Connectors: add `mcp-server-connectors` module** — pom.xml with full dependency tree + application.properties
4. **Test: JVM mode** — `java -jar`, stdio transport, verify tools/list + tools/call with ref providers
5. **Test: domain filtering** — explicit mode with subset of domains
6. **Test: provider activation** — configure a real provider (e.g. Google Calendar), verify switchover from ref
7. **Test: native mode** — GraalVM native build, verify startup + ref provider execution
8. **Distribution: container image** — Dockerfile for JVM and native variants
9. **Documentation: README** — langchain4j integration example, Claude Desktop config, provider setup

## 10. What this does NOT include

- **Build-time JSON manifests** — YAGNI. CDI runtime discovery handles tool catalog. No changes to `GraphQLResolverProcessor`.
- **Build-time domain filtering** — deferred. Runtime filtering at tools/list level is sufficient for v1. Build-time CDI elimination is a future optimisation.
- **Multi-tenant auth** — deferred. HTTP transport is local-only by default.
- **Google provider HTTP migration** — separate issue, not gated by this work
- **Tool schema versioning** — deferred, follow platform convention when established
- **Spring Boot standalone server** — `connectors-spring` coexists but this spec targets the Quarkus path only

## References

- casehubio/connectors#324 — epic issue
- casehubio/connectors#323 — implementation issue with architecture discussion
- `docs/audits/2026-11-19-native-image-readiness.md` — native image readiness audit
- `docs/protocols/connectors/mcp-tool-blocking-annotation.md` — @Blocking protocol for @Tool methods
- `docs/protocols/connectors/shared-http-client.md` — HttpHelper.CLIENT singleton protocol
- Platform `mcp-standalone-test` — proves standalone consumption without casehub-platform-api
- Platform `GraphQLModelScanner` — existing runtime @McpDomain bean discovery
- Platform `DynamicToolRegistrar` — existing tool registration via ToolManager
- Platform `mcp-core` — CDI-free domain model, schema builder (native-safe)
- MCP spec 2025-03-26 — Streamable HTTP transport
- Design review: `decisions.md` D2, D3, D5, D9 superseded — single-process replaces native proxy + JVM backend
