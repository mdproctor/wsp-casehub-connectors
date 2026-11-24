# Handoff — connectors

## Session summary

**Landed:** #208 — `mobile-numbrrs` module (Numbrrs MobilePlatform provider, Sms read-only + NumberInfo, HMAC-SHA256 webhooks, 17 tests). Code reviewed, docs updated, blog written, squashed, merged to main.

**In progress:** #356 — ConnectorLifecycle SPI Phase 1. Batch 1 (Foundation) complete:
- `ConnectorManifest` record with nested types, YAML parsing, forward-compatibility (7 tests)
- `ConnectorManifestLoader` with classpath + file-system merge cascade (5 tests)
- `ProviderCredentialFacade` interface + `InMemoryCredentialFacade` stub (4 tests)
- 12 architectural decisions captured (D1-D12) in `specs/issue-356-connector-lifecycle/decisions.md`
- Spec written, reviewed (3 rounds, 20 issues), committed
- Implementation plan written (5 batches, 11 tasks), committed and injected into queue

**Filed:** platform#966 (ProviderCredentialFacade — platform dependency), soredium#635 (work-start branch creation bug)

## Cross-repo state

| Repo | Branch | Status |
|------|--------|--------|
| connectors | issue-356-connector-lifecycle | 3 commits (Batch 1 foundation types) |
| wsp-connectors | issue-356-connector-lifecycle | decisions, spec, plan, queue |

## Resume

Batch 2: Build Extension — `ConnectorRegistry`, Quarkus `SyntheticBeanBuildItem` build steps. Plan: `plans/2026-11-24-connector-lifecycle-phase1.md`. 4 batches remaining.

## References

- Spec: `specs/issue-356-connector-lifecycle/2026-11-24-connector-lifecycle-phase1-design.md`
- Decisions: `specs/issue-356-connector-lifecycle/decisions.md`
- Plan: `plans/2026-11-24-connector-lifecycle-phase1.md`
- Blog: `blog/2026-11-24-mdp01-real-sims-for-real-testing.md`
