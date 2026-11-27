# Decisions — #378 TrueLayer PSD2 Consent Flow

## D1: actorId-aware FamilyConnectionProvider

**Choice:** Make `FamilyConnectionProvider.getAccessToken()` respect the `actorId` parameter — when non-null and the flow is `per-user: true`, delegate to a `UserTokenResolver` instead of the tenant credential facade
**Alternatives:**
- Composite keys in ProviderCredentialFacade (`truelayer:user123`) — mechanically possible but conflates tenant and user lifecycles, loses `removeExpiredBefore()` cleanup, domain-rich `StoredConsent`, JPA persistence
- Separate ServiceConnectionProvider per provider — duplicates boilerplate, loses manifest-driven token resolution
**Rationale:** `ServiceConnectionProvider.getAccessToken(actorId, provider, tenancyId)` was designed for per-user resolution from day one. The actorId parameter exists on every call site. `FamilyConnectionProvider` just never used it. Closing this gap unifies tenant and user token resolution under one provider without conflating storage.
**Trade-offs:** Adds per-user routing complexity to FamilyConnectionProvider. Mitigated by clean delegation to UserTokenResolver SPI.
**Sources:** `ServiceConnectionProvider` interface (actorId parameter); `FamilyConnectionProvider.getAccessToken()` (ignores actorId); all Google/Microsoft/TrueLayer callers (pass userId as actorId)
**Exploration:** deep-analysis
**Status:** captured

## D2: Keep TrueLayerConsentService as-is

**Choice:** TrueLayerConsentService stays as the runtime consent implementation; no extraction of shared consent base with OAuthConsentService
**Alternatives:**
- Extract shared OAuth flow base — reduces small duplication but couples tenant consent (PKCE, per-tenant storage) with user consent (no PKCE, per-user storage, 90-day PSD2 expiry)
**Rationale:** Behavioral differences are substantial: PKCE vs no PKCE, per-user vs per-tenant storage, PSD2 90-day consent lifecycle vs standard token expiry. The OAuth POST mechanics are the only shared code (~20 lines). A shared abstraction would need to abstract away the interesting parts.
**Trade-offs:** OAuth POST code duplicated between the two services. Acceptable — the duplicate is small and stable.
**Sources:** `OAuthConsentService.java` (PKCE flow); `TrueLayerConsentService.java` (no PKCE, per-user ConsentTokenStore)
**Exploration:** quick
**Depends on:** D1 (FamilyConnectionProvider delegates to TrueLayerConsentService via UserTokenResolver)
**Status:** captured

## D3: Standalone manifest with inline OAuth (no family)

**Choice:** TrueLayer uses inline OAuth config in its connector manifest, not a family manifest
**Alternatives:**
- Create a `truelayer` family manifest — adds indirection for a single-module provider with one client-id; family manifests exist to share credentials across multiple modules
**Rationale:** Families exist for shared app registrations (Google: 4 modules, Microsoft: 4 modules). TrueLayer is a single provider — a family manifest adds no value.
**Trade-offs:** If a second TrueLayer module appears (e.g. truelayer-payouts), would need migration to a family. Unlikely given TrueLayer's single-API design.
**Sources:** `google.yaml`, `microsoft.yaml` (multi-module families); existing `truelayer` manifest (standalone)
**Exploration:** quick
**Status:** captured

## D4: per-user flag on FlowDeclaration

**Choice:** Add `boolean perUser` to `FlowDeclaration` (default false). When true, `FamilyConnectionProvider` routes through `UserTokenResolver` instead of `ProviderCredentialFacade` for token resolution.
**Alternatives:**
- Infer per-user from grant type — authorization_code doesn't always mean per-user (Google/Microsoft delegated tokens are currently tenant-scoped)
- Always route through UserTokenResolver for authorization_code — breaks existing Google/Microsoft flows that store tokens in the tenant facade
**Rationale:** Explicit is better than implicit. PSD2 consent is inherently per-user; Google/Microsoft delegated tokens are currently per-tenant. The flag lets providers opt in without breaking existing flows. Future migration of Google to per-user is possible by flipping the flag.
**Trade-offs:** New field on FlowDeclaration; manifests must declare it explicitly for per-user flows.
**Sources:** `TrueLayerConsentService` (per-user storage); `FamilyConnectionProvider` (per-tenant storage); `FlowDeclaration` record (from #376)
**Exploration:** quick
**Depends on:** D1 (actorId-aware routing needs a signal for when to delegate)
**Status:** captured

## D5: UserTokenResolver CDI SPI

**Choice:** New `UserTokenResolver` interface with `providerId()` and `getAccessToken(userId)`. Provider modules implement it. `FamilyConnectionProvider` discovers beans at startup and delegates per-user requests.
**Alternatives:**
- FamilyConnectionProvider depends directly on ConsentTokenStore — couples the provider to a specific storage implementation; other providers might store per-user tokens differently
- Callback pattern (provider registers a lambda) — less discoverable than CDI, harder to test
**Rationale:** CDI SPI follows the existing pattern (Connector SPI, ChatPlatform SPI, etc.). TrueLayerConsentService already has the `getUserToken(userId)` method — it just needs to implement the interface. Clean separation: FamilyConnectionProvider knows how to route; provider modules know how to resolve.
**Trade-offs:** New SPI interface to maintain. Minimal — single method, stable contract.
**Sources:** `TrueLayerConsentService.getUserToken()` (existing per-user resolution); CDI SPI pattern used throughout the project
**Exploration:** quick
**Depends on:** D1, D4 (routing + per-user flag trigger delegation to this SPI)
**Status:** captured
