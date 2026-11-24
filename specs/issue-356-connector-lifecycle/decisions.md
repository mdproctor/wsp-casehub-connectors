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

## D9: Credential resolution — platform-level unified facade (REVISED)

**Choice:** Platform gains a `ProviderCredentialFacade` in `platform-api` that bridges the three existing credential mechanisms: `StaticCredentialStore` (API keys, bot tokens), `CredentialResolver` (config/vault/env), and `ServiceConnectionProvider` (OAuth tokens). Connectors and all other SPI providers declare credential fields in their manifest; the facade resolves them. No new credential store in connectors — connectors calls platform.
**Alternatives:**
- ~~ConnectorCredentialStore in connectors-api~~ — SUPERSEDED. This duplicated platform concerns.
- StaticCredentialStore only — doesn't cover config-based credentials
- MicroProfile Config only — doesn't cover stored credentials or OAuth
**Rationale:** Platform already owns all three credential mechanisms. A facade that bridges them is a platform concern, not a connector concern. Agents and users need a single interface to discover what's configured, what's missing, and how to set up missing credentials — regardless of the underlying mechanism. This serves agents, CLI, and future Mosaic UI equally.
**Trade-offs:** Cross-repo work (platform + connectors). Platform must expose enough for connectors to declare needs and receive resolved credentials without knowing which backend provided them.
**Depends on:** D1 (replace and unify), D8 (synthetic beans consume the platform facade)
**Sources:** StaticCredentialStore API, CredentialResolver API, ServiceConnectionProvider API — all in platform-api authn/credentials packages
**Exploration:** quick
**Status:** revised

## D10: Platform/connectors boundary — what lives where

**Choice:** Platform owns generic credential resolution, auth patterns, health check primitives, and the agent-facing discovery/setup API surface. Connectors owns manifest schema and discovery, synthetic bean production, family groupings (Google 4-connector, Microsoft 3+), and setup playbooks for connector-specific init steps (webhook registration, phone verification flows).
**Alternatives:**
- Everything in connectors — duplicates platform credential infrastructure
- Everything in platform — platform doesn't know about connector families, manifest semantics, or which beans to produce
**Rationale:** The credential facade and agent-facing status/setup tools are needed by any SPI provider (voice, calendar, commerce), not just connectors. The connector-specific knowledge (families, what beans a manifest produces, how Telegram webhooks work) is domain knowledge that belongs in connectors.
**Trade-offs:** Phase 1 requires coordinated work across two repos. Platform delivers the facade; connectors delivers the manifest and lifecycle manager that consumes it.
**Depends on:** D9 (credential facade in platform)
**Sources:** Platform authn package survey, connectors *Beans.java patterns
**Exploration:** quick
**Status:** captured

## D11: Init steps — playbook reference, not manifest-inline

**Choice:** Connectors that need setup steps beyond credential wiring declare an optional `setup-playbook` path in the manifest. The playbook is a standard YAML playbook file (same infrastructure as `voice-pipeline-steps`, `stt-transcribe`, etc.). Cross-connector flows (Signal phone verification via Numbrrs SMS) are playbook steps that call other platform connectors. The lifecycle manager runs the playbook when a connector is first configured or when setup is explicitly requested.
**Alternatives:**
- Manifest-inline steps — simpler for small flows but doesn't scale for multi-step cross-connector orchestration
- Programmatic ConnectorSetup SPI — maximum flexibility but back to bespoke Java per connector, contradicts D1
**Rationale:** Playbook infrastructure already exists, already handles step sequencing, already has integration tests. Reusing it means no new execution model. Cross-connector references are natural — a playbook step can call any MCP tool or platform operation. Signal's setup playbook calls `mobile_send_sms` (Numbrrs), waits for the verification SMS, extracts the code, and calls Signal's register endpoint.
**Trade-offs:** Playbooks need step types for health-check and wait-for-SMS that may not exist yet. The setup playbook must be idempotent — running it again on an already-configured connector should be a no-op, not a re-registration.
**Depends on:** D2 (YAML format), D10 (setup playbooks are connector-specific, live in connectors)
**Sources:** Existing playbook infrastructure in connectors/playbooks, voice-pipeline-steps pattern
**Exploration:** quick
**Status:** captured

## D12: Manifest conventions — follow voice-config/agent-config pattern

**Choice:** `connector-manifest.yaml` follows the established manifest conventions: kebab-case keys, `@JsonProperty("kebab-name")` mapping to camelCase Java record fields, `YamlMappers.create()` for parsing, record-based Jackson deserialization, prioritised merge cascade (classpath → /etc/casehub → ~/.casehub → project dir → profile variant). Top-level structure uses named sections consistent with voice-config (`providers`, `voices`, `defaults`) and agent-config (`providers`, `models`, `aliases`, `defaults`, `pools`).
**Alternatives:**
- camelCase (used by seed-catalog.yaml, test-endpoints.yaml) — inconsistent with the dominant pattern
- Flat properties (like yaml-services.yaml) — too simple for the manifest's hierarchical structure
**Rationale:** Survey of 14 YAML patterns across the platform shows kebab-case + record-based deserialization + merge cascade as the dominant convention. Voice-config and agent-config are the closest structural analogues. Consistency means anyone who has worked with voice-config or agent-config will recognise the connector-manifest pattern immediately.
**Trade-offs:** None — this is a conformance decision, not a trade-off.
**Sources:** voice-config.yaml (VoiceManifest, VoiceManifestLoader), agent-config.yaml (AgentManifest, AgentManifestLoader), platform YAML convention survey
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
