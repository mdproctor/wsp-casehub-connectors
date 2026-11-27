# Decisions — #376 Microsoft OAuth Family Manifest

## D1: Named flows within family manifest

**Choice:** Extend `FamilyManifest` with a `flows` map — each flow declares grant type, scopes, and optional token-url override
**Alternatives:**
- Separate sub-families (`microsoft-delegated`, `microsoft-app`) — splits shared credentials across manifests, violates single-app-registration reality
- Keep family single-flow, treat application/bot as standalone manifests — loses the credential-sharing benefit that families exist to provide
**Rationale:** Microsoft uses one Azure AD app registration with one client-id/client-secret for three distinct OAuth flows (delegated, application, bot). Named flows model this directly.
**Trade-offs:** Adds complexity to `FamilyManifest` model; families without multiple flows carry unused structure (mitigated by backward compat — omitting `flows` means single implicit flow)
**Sources:** Microsoft Bot Framework REST Connector Authentication docs; `TeamsChatPlatform.java` (three provider strings); `FamilyManifest.java`; `microsoft.yaml`
**Exploration:** quick
**Status:** captured

## D2: Per-module flow references

**Choice:** Connector manifests declare `auth.flows: [delegated]` or `auth.flows: [delegated, application, bot]` to specify which family flows they use
**Alternatives:**
- Implicit from capabilities — infer flow from what the module does (e.g. "sends messages" → application) — too magical, breaks when capabilities span flows
- Single flow per manifest, multiple manifests per module — overcomplicates module structure
**Rationale:** Explicit is better. Teams needs three flows; calendar/email/document need one. The manifest should say so.
**Trade-offs:** Modules must know their flow names, coupling manifest to family structure
**Sources:** Google connector manifests (`auth.family: google`); `ConnectorManifestLoader.enrichWithFamily()`
**Exploration:** quick
**Depends on:** D1 (named flows must exist for modules to reference them)
**Status:** captured

## D3: Convention-based provider string → family:flow mapping

**Choice:** `FamilyConnectionProvider` maps provider strings by convention: `{familyId}` → primary flow, `{familyId}-{flowName}` → named flow
**Alternatives:**
- Manifest-declared provider keys (`provider-key: microsoft-app` in each flow) — explicit but adds a field that must stay in sync with code
**Rationale:** Teams already uses `"microsoft"`, `"microsoft-app"`, `"microsoft-bot"` — the convention matches existing code with zero changes to platform modules
**Trade-offs:** Convention is implicit; a typo in a provider string silently fails to match. Mitigated by `DependencyResolver` surfacing unmet requirements at startup.
**Sources:** `TeamsChatPlatform.java` (`resolveDelegatedToken`, `resolveApplicationToken`); `BotFrameworkClient.java` (`resolveToken`)
**Exploration:** quick
**Depends on:** D1 (flow names are the suffix in the convention)
**Status:** captured

## D4: FamilyConnectionProvider handles client_credentials directly

**Choice:** `FamilyConnectionProvider` detects `grant: client_credentials` flows and POSTs directly to the token endpoint with client-id/client-secret, bypassing `OAuthConsentService`
**Alternatives:**
- Extend `OAuthConsentService` to support client_credentials — adds a code path to a class designed around user-redirect flows, muddying its responsibility
**Rationale:** client_credentials has no user consent redirect — it's a simple POST. OAuthConsentService is built around PKCE authorization_code flow. Different grant types, different code paths.
**Trade-offs:** Token acquisition logic lives in two places (OAuthConsentService for delegated, FamilyConnectionProvider for application). Acceptable because the flows are fundamentally different.
**Sources:** `OAuthConsentService.java` (PKCE flow); `FamilyConnectionProvider.java` (token resolution + refresh)
**Exploration:** quick
**Depends on:** D1 (flow grant type determines code path)
**Status:** captured

## D5: Proceed without waiting for #373

**Choice:** #373 (PersistentCredentialFacade) is already implemented — no dependency blocker
**Alternatives:** N/A — this was a constraint check, not a design choice
**Rationale:** `PersistentCredentialFacade` is fully coded, tested, and wired up. Issue was open but work was done. Closed it.
**Sources:** `PersistentCredentialFacade.java`; `DefaultFacadeCreator.java`
**Exploration:** quick
**Status:** captured
