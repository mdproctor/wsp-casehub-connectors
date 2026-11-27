# HANDOFF — casehub-connectors (#378)

## Last Session

Implemented #378 (truelayer-psd2-consent) — all 4 tasks complete across 3 batches on branch `issue-378-truelayer-psd2-consent`. Code-reviewed, one real bug found and fixed.

**What was built:**

- **`UserTokenResolver` CDI SPI** — new interface in `connectors-api` with `providerId()` and `getAccessToken(userId)`. Provider modules implement it for per-user token resolution.
- **`per-user: true` on `FlowDeclaration`** — nullable `Boolean` field with `isPerUser()` accessor defaulting to false. Deserialized from `per-user` in YAML manifests.
- **`FamilyConnectionProvider` actorId-aware routing** — 3-arg constructor accepting `Map<String, UserTokenResolver>`, CDI discovery via `buildResolverMap()` using `!instances.isUnsatisfied()`, guard clause at top of `getAccessToken`: when actorId is non-null and flow is per-user, delegates to resolver.
- **`TrueLayerConsentService implements UserTokenResolver`** — exposes existing `getUserToken(userId)` through the SPI. No changes to internals or `ConsentTokenStore`.
- **`DependencyResolver` per-user awareness** — 4-arg constructor with `Set<String> availableUserResolvers`. Per-user authorization_code flows check resolver availability instead of stored access-token.
- **`truelayer.yaml` family manifest** — new family manifest in `connectors-api/src/main/resources/META-INF/families/` declaring the per-user delegated flow.
- **Updated connector manifest** — `bank-truelayer/connector-manifest.yaml` now references `family: truelayer` with `flows: [delegated]`.
- **Review fix** — `Instance.isResolvable()` → `!instances.isUnsatisfied()` for multi-bean CDI discovery.

## Branch State

- Project + workspace: on `issue-378-truelayer-psd2-consent`
- 5 commits for #378 (4 feature + 1 review fix), plus earlier #376 commits
- Working tree clean
- #378 marked done in .queue, #354 is active (not started)

## Immediate Next Step

Start #354 (platform-registration-spi) — unknown scale, needs brainstorming/design first. Or run `work end` to land #378 and earlier done items.

## Known Issues

- **Integration test failure** (pre-existing) — `ConnectorLifecycleProcessor#registerSyntheticBeans` throws "Local name must not be null". Unrelated to #378. Build succeeds with `-pl '!integration-tests'`.
- **MCP server test failure** (pre-existing, #382) — `McpServerStartupTest` class discovery failure.

## References

- #378 plan: `plans/2026-11-27-truelayer-psd2-consent.md`
- #378 decisions: `specs/issue-378-truelayer-psd2-consent/decisions.md`
- Diary: `_notes/2026-11-27-mdp03-three-flows-one-family.md` (updated with #378 implementation)
