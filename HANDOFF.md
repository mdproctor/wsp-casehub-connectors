# Handoff — connectors

## Session 2026-11-25

### Completed

**#363 — Integration test for manifest-driven synthetic bean wiring**
- `@QuarkusTest` in `integration-tests` with real Quarkus app boot
- Verifies `ConnectorRegistry`, `NumbrrsMobilePlatform` injection, `ProviderCredentialFacade` resolution
- Added `quarkus-junit`, `connector-lifecycle`, `mobile-numbrrs`, `chat-signal` dependencies
- `TestProfile` excludes unrelated CDI beans for clean container boot

**#365 — Manifest requires/provides with backward-chaining dependency resolution**
- `CapabilityDeclaration(capability, sub)` and `RequirementDeclaration(capability, sub, reason)` in `ConnectorManifest`
- `DependencyResolver` — static graph, sub-capability matching, `ResolutionOption` sealed hierarchy (ProviderOption, ManualOption)
- Signal manifest: `requires: sms[receive]`, Numbrrs manifest: `provides: sms[receive], number-info`
- Signal registration playbook: `connector-status: numbrrs` → `require-capability: sms[receive]`

**#366 — require-capability playbook step runner**
- `RequireCapabilityPlugin` `@Plugin` step type — resolves capabilities via `DependencyResolver`
- Returns success with provider ID when satisfied, failure with resolution options when not
- `DependencyResolver` registered as synthetic CDI bean by `ConnectorLifecycleProcessor`
- `DependencyResolverCreator` builds resolver from classpath manifests at runtime

### Queue

Empty. All three items landed.

### Next session must know

- Backward-chaining setup dependencies are a universal pattern: SMS + email as bootstrap primitives. A human captcha pass-through may exist in neocortex corpus — worth investigating for `ManualOption` flows.
- Sub-capabilities use convention-based strings, no enum. Unqualified provider (empty sub) means full capability — satisfies any qualified requirement.
- The `require-capability` step returns `Result.failed()` with `category: "unmet-requirement"` and resolution options in `detail.options` — the playbook runner needs to handle this category specially (present options to user, trigger recursive setup).

## References

- Spec: `specs/issue-365-manifest-requires/2026-11-25-manifest-requires-design.md`
- Decisions: `specs/issue-365-manifest-requires/decisions.md`
- Plan: `plans/2026-11-25-manifest-requires.md`
- Blog: `blog/2026-11-26-mdp01-what-do-you-need.md`
- Previous: `blog/2026-11-25-mdp01-the-manifest-that-replaced-the-beans.md` (#356)
