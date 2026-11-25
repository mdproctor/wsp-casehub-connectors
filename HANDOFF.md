# Handoff — connectors

## Session summary

**In progress:** #356 — ConnectorLifecycle SPI Phase 1. Batches 1-3 complete (10 commits on branch):

**Batch 1 (Foundation):**
- `ConnectorManifest` record with YAML parsing, forward-compatibility
- `ConnectorManifestLoader` with classpath + file-system merge cascade
- `ProviderCredentialFacade` interface + `InMemoryCredentialFacade` stub

**Batch 2 (Build Extension):**
- `ConnectorRegistry` + `ConnectorStatus` — tracks manifest-declared connectors with lifecycle state
- `connector-lifecycle` module — Quarkus build extension (`ConnectorLifecycleProcessor`) with `SyntheticBeanBuildItem` registration, `ConnectorBeanCreator` for manifest-driven construction, default `ProviderCredentialFacade`

**Batch 3 (Migrations):**
- Numbrrs: manifest + String constructor, `NumbrrsMobileBeans` deleted
- Telegram: manifest + String constructor, CDI annotations stripped, timing-safe webhook secret fix (`MessageDigest.isEqual`), `ChatTelegramBeans` deleted
- Signal: manifest + String constructor, CDI annotations stripped, `ChatSignalBeans` deleted, CDI wiring test removed (integration-level concern)

**Review fix:** Removed `@PostConstruct` from `TelegramChatPlatform.init()` and `SignalChatPlatform.init()` — synthetic beans invoke `@PostConstruct` after construction, so the manifest-driven constructors calling `init()` would cause double-init.

**Filed:** #362 (move `@ConnectorBean` to `connector-lifecycle` with `@Qualifier`), #363 (integration test for manifest-driven wiring)

## Cross-repo state

| Repo | Branch | Status |
|------|--------|--------|
| connectors | issue-356-connector-lifecycle | 10 commits (Batches 1-3 + review fix) |
| wsp-connectors | issue-356-connector-lifecycle | decisions, spec, plan, queue, blog, issues |

## Resume

Batch 4: MCP Tools — `ConnectorLifecycleOperations` under `@McpDomain("connectors/lifecycle")` with `connector_discover`, `connector_status`, `connector_setup`, `connector_health`. Plan: `plans/2026-11-24-connector-lifecycle-phase1.md`. 2 batches remaining.

**Key context for next session:**
- `connector-lifecycle` is a single-module Quarkus extension (both runtime and deployment in one module). Registered via `quarkus-extension.properties`. The `quarkus-extension-processor` generates `quarkus-build-steps.list`.
- `@ConnectorBean` annotation currently lacks `@Qualifier` — `connectors-api` is framework-neutral. #362 will move it to `connector-lifecycle`.
- Synthetic beans use `@Default` qualifier (no `@Named`). `Instance<ChatPlatform>` iteration works for platform service discovery.
- Tests use `QuarkusUnitTest` with explicit application root containing build step classes and `quarkus-build-steps.list`.

## References

- Spec: `specs/issue-356-connector-lifecycle/2026-11-24-connector-lifecycle-phase1-design.md`
- Decisions: `specs/issue-356-connector-lifecycle/decisions.md`
- Plan: `plans/2026-11-24-connector-lifecycle-phase1.md`
- Blog: `blog/2026-11-25-mdp01-the-manifest-that-replaced-the-beans.md`
