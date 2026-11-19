# MCP Server Packaging — Decisions

## D1: Server composition model

**Choice:** Composable — single server loads tools from multiple domain projects via classpath composition
**Alternatives:**
- Separate only — one server per domain project, no cross-domain. Simpler but requires running multiple servers
- Combined only — one server with everything. Simplest packaging but bloated for single-domain use
**Rationale:** Platform proxy is domain-agnostic. A connectors-only, neocortex-only, or combined server are all different dependency sets in the config module's pom.xml. No architectural difference — just what lands on the classpath.
**Trade-offs:** Config module authors must understand which @McpDomain modules to include. Tool name collisions across domains are possible (mitigated by domain-prefixed tool names).
**Sources:** Platform mcp module architecture (GraphQLModelScanner auto-discovers all @McpDomain beans on classpath)
**Exploration:** quick
**Status:** captured

## D2: Proxy-to-backend transport

**Choice:** Unix domain socket
**Alternatives:**
- HTTP over localhost — simple, debuggable but opens a port
- stdin/stdout pipe — tight coupling, zero network surface but harder to debug
**Rationale:** Local IPC only, no network exposure. Fast, secure. Proxy and backend are co-located by design.
**Trade-offs:** Not portable to Windows (named pipes differ). Container deployments need shared socket volume.
**Sources:** Standard sidecar architecture pattern
**Exploration:** quick
**Status:** captured

## D3: Backend lifecycle management

**Choice:** Proxy manages backend — single entry point, proxy starts/monitors/restarts the JVM backend
**Alternatives:**
- Independent processes — user starts both separately. More flexible for k8s but harder for local use
- Both modes — default managed, flag for independent. Covers both but adds config surface
**Rationale:** Single command to start. The proxy IS the server; the backend is an implementation detail. End users shouldn't need to know about the two-process architecture.
**Trade-offs:** Proxy becomes a process supervisor. Must handle backend crashes, restarts, and clean shutdown. Container deployments that want independent processes need workarounds.
**Sources:** #323 architecture discussion (native proxy + JVM backend)
**Exploration:** quick
**Status:** captured

## D4: Tool catalog source

**Choice:** Per-module APT-generated manifests aggregated at startup — each @McpDomain module APT-generates its own tool manifest (e.g. `META-INF/casehub-mcp/<domain>.json`) containing tool names and JSON schemas. The proxy at startup scans `ClassLoader.getResources("META-INF/casehub-mcp/")`, reads all fragments, merges into a single tool catalog. Pure classpath resource loading — no reflection, native-image-safe, sub-millisecond aggregation.
**Alternatives:**
- Single build-time catalog baked into native image — fast but requires per-combination native builds, defeating runtime composability
- First-call fetch from backend — simpler but first tools/list blocks on JVM startup
- Embedded ref dispatch — ref impls in native image for instant execution, not just catalog. More complex native build
**Rationale:** Each domain module is independently responsible for its own catalog fragment via APT. The proxy is domain-agnostic — it reads whatever manifests land on the classpath. A connectors-only, neocortex-only, or combined distribution is just different JARs in the same classpath. No per-combination native build needed. Jandex + McpDomainJandexScanner already proves the scanning path works.
**Trade-offs:** Catalog is frozen at module build time. Dynamic tool registration (providers adding tools at runtime) requires backend readiness + notifications/tools/list_changed. APT must run in each domain module's build.
**Sources:** mcp-standalone-test proves Jandex scanning works without casehub-platform-api; McpDomainJandexScanner in graphql-spring-generator; ServiceLoader META-INF pattern
**Exploration:** quick
**Status:** captured

## D5: Pre-backend-ready tool call behaviour

**Choice:** Queue and wait — proxy holds requests until backend reports readiness for the domain, then forwards
**Alternatives:**
- Ref fallback in proxy — ref impls in native image serve calls during warmup. Zero-latency but complex native build
- Error with retry hint — honest but poor UX, most MCP clients don't handle retry hints
**Rationale:** MCP has no timeout spec — client waits. Simple, correct, no fake data. First-call latency is JVM startup (~2-5s), which is negligible since the LLM is typically still processing.
**Trade-offs:** First tool call after startup has 2-5s latency. If backend crashes during queue, requests are lost (proxy should return error after timeout).
**Sources:** #323 architecture — warmup discussion
**Exploration:** quick
**Status:** captured

## D6: Client-facing MCP transports

**Choice:** Both stdio and SSE — flag to select (--transport=stdio default, --transport=sse for remote)
**Alternatives:**
- stdio only — simplest, covers Claude Desktop and langchain4j but no remote access
- SSE only — network-first but many local MCP clients expect stdio
**Rationale:** Covers all use cases. stdio for local agent frameworks, SSE for web/container deployments.
**Trade-offs:** Two protocol implementations in the proxy. SSE needs HTTP server in the native image (adds to binary size).
**Sources:** MCP spec defines both stdio and SSE as standard transports
**Exploration:** quick
**Status:** captured

## D7: Explicit mode domain filtering

**Choice:** Proxy filters the tool catalog at runtime; backend loads all providers on the classpath
**Alternatives:**
- Both filter — backend skips excluded domains too. Saves resources but adds coordination complexity
- Backend filters only — proxy transparent, but defeats instant-catalog goal
**Rationale:** Backend doesn't need to know about explicit mode. Proxy's build-time catalog includes all tools; runtime config property filters what tools/list exposes. Simple separation of concerns.
**Trade-offs:** Slight resource waste — backend starts providers for domains the proxy won't expose. Acceptable because startup time is dominated by the slowest provider regardless.
**Sources:** #323 explicit mode discussion
**Exploration:** quick
**Status:** captured
