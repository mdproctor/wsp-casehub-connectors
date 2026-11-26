# Handoff — connectors

## Session 2026-11-25

### Completed

**#363 — Integration test for manifest-driven synthetic bean wiring**
- `@QuarkusTest` in `integration-tests` with real Quarkus app boot
- Verifies `ConnectorRegistry`, `NumbrrsMobilePlatform` injection, `ProviderCredentialFacade` resolution

**#365 — Manifest requires/provides with backward-chaining dependency resolution**
- `CapabilityDeclaration(capability, sub)` and `RequirementDeclaration(capability, sub, reason)` in `ConnectorManifest`
- `DependencyResolver` — static graph, sub-capability matching, `ResolutionOption` sealed hierarchy
- Signal: `requires: sms[receive]`, Numbrrs: `provides: sms[receive], number-info`
- Signal playbook: `connector-status: numbrrs` → `require-capability: sms[receive]`

**#366 — require-capability playbook step runner**
- `RequireCapabilityPlugin` `@Plugin` step type via `DependencyResolver` CDI bean
- Returns success with provider ID when satisfied, failure with resolution options when not

**#367 — provides declarations for Telegram and Signal**
- Telegram: `provides: chat[messaging, threading, discovery, reactions, members, presence, commands]`
- Signal: `provides: chat[messaging, discovery, members, reactions, channel-management, member-management]`

**#368 — Email as bootstrap primitive**
- Unit tests proving email resolves identically to SMS in the dependency graph
- Dual-primitive chain test: service requiring both sms[receive] and email[receive]

### Queue

Empty. All items landed.

### Next work — setup chain

Queued in platform repo:
- **platform#986** — playbook runner handles `unmet-requirement` resolution options (bottleneck)
- **platform#987** — ManualOption flow for user-provided capability resolution

### Next session must know

- SMS + email are bootstrap primitives — with just these two, you chain into every other service
- Human captcha pass-through may exist in neocortex corpus — investigate for `ManualOption`
- `require-capability` returns `Result.failed(category: "unmet-requirement")` with options in `detail` — platform playbook runner needs to handle this

## References

- Spec: `specs/issue-365-manifest-requires/2026-11-25-manifest-requires-design.md`
- Blog: `blog/2026-11-26-mdp01-what-do-you-need.md`
