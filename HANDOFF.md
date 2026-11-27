# HANDOFF — casehub-connectors (#376, #378)

## Last Session

Completed #376 (microsoft-oauth-family) implementation on branch `issue-376-microsoft-oauth-family`. Designed and planned #378 (truelayer-psd2-consent) — spec and plan written, ready for implementation.

**#376 — What was built:**

- **Multi-flow `FamilyManifest`** — `FlowDeclaration` record with `GrantType` enum (`AUTHORIZATION_CODE`, `CLIENT_CREDENTIALS`), optional `token-url` override, and `perUser` flag. Families can now declare named flows.
- **`microsoft.yaml` updated** — three flows: `delegated` (authorization_code, 6 Graph scopes), `app` (client_credentials, graph.microsoft.com), `bot` (client_credentials, botframework.com with custom token-url).
- **Connector manifests created** — calendar-microsoft, document-microsoft, chat-teams (new); email-microsoft cleaned up. Each declares which flows it uses.
- **Convention-based provider string resolution** — `resolveFamily()` maps `"microsoft"` → delegated, `"microsoft-app"` → app, `"microsoft-bot"` → bot. Existing Teams code unchanged.
- **`client_credentials` grant support** — `acquireApplicationToken()` in `FamilyConnectionProvider` POSTs to token endpoint with shared client-id/client-secret. Auto-acquires when missing or expired.
- **`DependencyResolver` multi-flow** — checks each declared flow independently. `FamilyClientCredentialsOption` added to sealed `ResolutionOption`. Authorization_code flows check for access-token; client_credentials flows check for client-id + client-secret.
- **`OAuthConsentService` flow scopes** — reads scopes from the delegated flow when present, falls back to top-level `oauth.scopes`.
- **Review fixes** — `resolveTokenUrl()` for `{tenant-id}` template resolution (was missing in both `acquireApplicationToken` and `doRefresh`); `delegatedScopes()` null safety.

**#378 — Design and plan complete, not implemented:**

- **UserTokenResolver CDI SPI** — new interface for per-user token resolution. `TrueLayerConsentService` implements it.
- **`per-user: true` on `FlowDeclaration`** — signals `FamilyConnectionProvider` to delegate to `UserTokenResolver` instead of tenant facade.
- **actorId-aware `FamilyConnectionProvider`** — when `actorId` is non-null and flow is per-user, delegates to discovered `UserTokenResolver` beans.
- **Plan:** 4 tasks, 3 batches at `/Users/mdproctor/claude/public/casehub/connectors/plans/2026-11-27-truelayer-psd2-consent.md`

## Immediate Next Step

Execute the #378 plan — 4 tasks across 3 batches. The plan is detailed with TDD steps and code. Start with `work continue`.

## Branch State

- Project repo: on `issue-376-microsoft-oauth-family` (needs alignment — workspace is on `issue-378-truelayer-psd2-consent`)
- Both branches have clean working trees
- #376 implementation committed but not landed (awaiting work-end)
- #378 branch has spec and plan committed, no implementation yet

## Known Issues

- **MCP server test failure** (pre-existing, #382) — `McpServerStartupTest` class discovery failure. Unrelated to #376/#378. Build succeeds with `-pl '!mcp-server-connectors'`.
- **Spec naming drift** (#376) — design spec says "application" for the flow name, implementation uses "app" (to match provider string convention `microsoft-app`). Spec not updated. Code is correct.

## References

- #376 design spec: `docs/specs/issue-376-microsoft-oauth-family/2026-11-27-microsoft-oauth-family-design.md`
- #376 decisions: `work/specs/issue-376-microsoft-oauth-family/decisions.md`
- #378 design spec: `docs/specs/issue-378-truelayer-psd2-consent/2026-11-27-truelayer-psd2-consent-design.md`
- #378 decisions: `work/specs/issue-378-truelayer-psd2-consent/decisions.md`
- #378 plan: `plans/2026-11-27-truelayer-psd2-consent.md`
- Diary: `docs/blog/2026-11-27-mdp03-three-flows-one-family.md`
