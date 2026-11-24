# ConnectorLifecycle Decisions

## D1: Scope — replace existing credential patterns, not layer on top

**Choice:** Replace and unify all three existing patterns (StaticCredentialStore, ServiceConnectionProvider, @ConfigProperty) with a single declarative model
**Alternatives:**
- Layer on top — less risk but perpetuates three parallel patterns
- Replace new / adapt old — gradual but results in long-lived adapters
**Rationale:** Onboarding friction is the primary problem. A unified model eliminates the need to understand three different credential patterns. The platform already has 15+ connectors using 3+ patterns — the complexity cost of maintaining all of them grows with every new connector.
**Trade-offs:** Touches every connector module. Must be phased to avoid a single massive migration.
**Sources:** Explorer survey of all connector init patterns (15+ modules)
**Exploration:** quick
**Status:** captured

## D2: Declaration format — YAML descriptor per module

**Choice:** Each connector module includes a `connector-manifest.yaml` declaring auth requirements, configuration, and init steps. Runtime discovers and processes them.
**Alternatives:**
- Java annotations on platform class — compile-time safety but less flexible for non-Java consumers
- Java SPI interface methods — most type-safe but couples declaration to implementation
**Rationale:** Consistent with existing `yaml-services.yaml` pattern. YAML is readable by humans and tooling (CLI, dashboard). Separates declaration from implementation — the manifest describes what's needed, the runtime handles how.
**Trade-offs:** No compile-time validation of manifest contents. Needs a schema and validation at startup.
**Sources:** yaml-services.yaml pattern, agent-config.yaml analogy
**Exploration:** quick
**Status:** captured

## D3: Connector families — shared authentication

**Choice:** Connectors declare a `family` in their manifest (e.g. `google`, `microsoft`). The lifecycle manager merges auth requirements across family members — one "Connect Google" provisions Calendar + Drive + Contacts + Gmail with unified scope consent.
**Alternatives:**
- Per-connector auth only — each connector manages its own credentials independently
**Rationale:** Google has 4 connectors sharing OAuth. Microsoft has 3+. Per-connector auth means 4 separate consent prompts for the same provider. Family-aware auth gives a single onboarding action per provider.
**Trade-offs:** Family members must be able to function independently (some users want Calendar but not Drive). The manifest needs optional/required scope distinction.
**Sources:** GoogleCredentialResolver CDI SPI (shared across 4 modules), Microsoft Graph shared client
**Exploration:** quick
**Status:** captured

## D4: Delivery — four-phase epic

**Choice:** Four phases, each delivering standalone value:
- **Phase 1 — Foundation + first movers:** Manifest schema, lifecycle manager, migrate Numbrrs/Telegram/Signal, `connector_status` MCP tool
- **Phase 2 — Credential unification:** Migrate all API-key connectors (Twilio, Shopify, Amazon, Amadeus), remove bespoke `*Beans.java` classes
- **Phase 3 — OAuth families:** Google family (4 connectors), Microsoft family (3+), GitHub, TrueLayer — unified scope consent per family
- **Phase 4 — Automated setup orchestration:** Cross-connector flows (Signal via SMS), webhook auto-registration, guided setup for external dashboards
**Alternatives:**
- Single-phase big-bang — too risky, blocks all connector work
- Two-phase (foundation + everything else) — Phase 2-4 split matters for review scope
**Rationale:** Each phase provides working value. Phase 1 proves the model on 3 diverse auth types (API key, token, phone verify). Phase 2 eliminates boilerplate. Phase 3 tackles the hardest auth pattern. Phase 4 adds the cross-connector orchestration that motivated the design.
**Exploration:** quick
**Status:** captured

## D5: Manifest schema — hybrid (family + per-module)

**Choice:** Family auth definitions live centrally in `connectors-api/families/`. Per-module `connector-manifest.yaml` files reference their family and declare scopes, capabilities, and init steps. Non-family connectors define auth inline.
**Alternatives:**
- Flat per-connector — each module fully self-contained, families discovered by implicit provider name matching (fragile)
- Grouped by family — one manifest per family (couples independent modules)
**Rationale:** Families need explicit grouping to merge OAuth scopes reliably. Per-module manifests preserve module independence. Central family definitions are the single source of truth for shared auth config (consent URLs, token endpoints, base scopes).
**Trade-offs:** Two file locations to maintain (family definitions + per-module manifests). Schema validation must cross-reference family references.
**Depends on:** D2 (YAML descriptor format)
**Exploration:** quick
**Status:** captured

## D6: Browser automation for credential capture

**Choice:** Require Playwright (or equivalent browser MCP) for automating OAuth consent flows, dashboard configurations, and credential capture. Semi-automated steps become fully automated.
**Alternatives:**
- Manual copy-paste — user grabs credentials from dashboards and enters them
- CLI-only — works for API keys but not for OAuth browser redirects
**Rationale:** The whole point of the lifecycle SPI is to eliminate onboarding friction. If the user still has to manually navigate Google consent pages or Slack dashboards, we've only moved the friction. Playwright handles OAuth redirects, dashboard form fills, and credential capture programmatically.
**Trade-offs:** Adds Playwright as a dependency for the setup flow (not for runtime). Users need a browser-capable environment for initial setup. Headless mode available for CI.
**Sources:** mcp__playwright__* tools already available in Claude Code sessions
**Exploration:** quick
**Status:** captured

## D7: Interface layering — MCP domain first, Mosaic UI later

**Choice:** Expose lifecycle management via `@McpDomain("connectors/lifecycle")` MCP tools as the primary API. CLI and config files for this iteration. Mosaic web UI in a future iteration, consuming the same MCP tools.
**MCP tools (Phase 1):**
- `connector_discover` — list available connectors and their manifest requirements
- `connector_setup` — run init steps for a connector (credential resolution, webhook registration, browser-driven OAuth)
- `connector_status` — what's configured, what's missing, what's healthy
- `connector_health` — runtime health check across all active connectors
**Alternatives:**
- REST API first — heavier, needs its own auth, duplicates MCP
- Mosaic UI first — wrong order; UI needs an API to call
**Rationale:** MCP is the universal interface — Claude sessions, CLI scripts, and Mosaic all consume it. Building MCP first means the UI is just a frontend over proven tools.
**Trade-offs:** No visual management until Mosaic integration. Acceptable for this iteration.
**Sources:** Existing `@McpDomain("connectors")` pattern (casehub_action, connectorStatus, connectorsReport)
**Exploration:** quick
**Status:** captured

## D8: CDI mechanism — Quarkus synthetic beans from manifest

**Choice:** A Quarkus build extension discovers `connector-manifest.yaml` files and registers synthetic CDI beans (via `SyntheticBeanBuildItem`) for each connector's platform class and inbound connector. The synthetic bean's creation function resolves credentials from a unified store, constructs the client, and constructs the platform/connector. All bespoke `*Beans.java` producers are deleted.
**Alternatives:**
- Config adapter — keep existing CDI beans, unify only the config source. Less invasive but perpetuates two parallel bean production styles.
- Annotated platform classes — `@ConnectorManifest` on Java classes instead of YAML. Compile-time safe but contradicts D2 (YAML descriptor).
**Rationale:** One mechanism for all connectors. Existing `@ApplicationScoped` classes (Telegram, Signal) become plain classes instantiated by the synthetic producer. `NumbrrsMobileBeans`, `ChatTelegramBeans`, `ChatSignalBeans` all disappear. Consistent with D1 (replace, not layer).
**Trade-offs:** Requires Quarkus build extension knowledge. Synthetic beans are less discoverable than explicit `@Produces` methods. Spring Boot deployment (`connectors-spring`) needs an equivalent mechanism.
**Depends on:** D1 (replace and unify), D2 (YAML descriptor)
**Sources:** Quarkus SyntheticBeanBuildItem API, existing credential wiring in NumbrrsMobileBeans/ChatTelegramBeans/ChatSignalBeans
**Exploration:** quick
**Status:** captured
