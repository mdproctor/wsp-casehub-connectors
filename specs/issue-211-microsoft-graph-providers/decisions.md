## D1: Raw HTTP via HttpHelper.CLIENT (not Microsoft Graph SDK)

**Choice:** Raw HTTP using `HttpHelper.CLIENT` singleton with manual JSON parsing and pagination
**Alternatives:**
- Microsoft Graph Java SDK — typed models and built-in pagination, but SDK owns auth lifecycle (conflicts with `ServiceConnectionProvider`), adds OkHttp transport that bypasses shared connection pool, and fluent API doesn't fit call-time credential passing
- Thin SDK wrapper (models only, raw HTTP for calls) — type safety without auth conflict, but added complexity for marginal benefit when Graph API responses are simple JSON
**Rationale:** `ServiceConnectionProvider.getAccessToken(actorId, "microsoft", tenancyId)` maps directly to a `Bearer` header on raw HTTP. The SDK requires per-user `GraphServiceClient` construction (heavy) or thread-local auth context (fragile). Graph API is clean REST with consistent JSON shapes — typed models add little value. Architecturally consistent with 4 of 5 existing shared clients (SlackBotClient, DiscordClient, GitHubClient, SignalClient).
**Trade-offs:** No typed response models — must define Java records for Graph response shapes. No built-in retry on token refresh — must handle 401 → re-acquire → retry manually.
**Sources:** shared-http-client protocol (PP-20260607-9794cb), credential-config-ownership protocol (PP-20260609-0c3e24), Google provider pattern (ServiceConnectionProvider in platform-api), gap analysis (docs/specs/2026-11-13-connector-gap-analysis.md)
**Exploration:** quick
**Status:** captured

## D2: ServiceConnectionProvider for credential resolution (not MicrosoftCredentialResolver)

**Choice:** Use existing `ServiceConnectionProvider` CDI SPI from `platform-api` with `provider = "microsoft"`
**Alternatives:**
- New `MicrosoftCredentialResolver` CDI SPI — more flexibility for Microsoft-specific auth flows (on-behalf-of), but duplicates existing infrastructure and diverges from Google provider pattern
**Rationale:** All Google modules use `ServiceConnectionProvider`. One unified SPI for all provider credential resolution. `@RequiresScopes(provider = "microsoft", scopes = {...})` declares needed OAuth2 scopes. Microsoft-specific auth details (Entra ID, tenant config) are implementation concerns of the `ServiceConnectionProvider` implementation, not the connector modules.
**Trade-offs:** If Microsoft auth requires fundamentally different flows not expressible via `getAccessToken()`, the SPI would need extension. Currently unlikely — delegated OAuth2 is the same flow for both Google and Microsoft.
**Sources:** platform-api ServiceConnectionProvider SPI, Google provider modules (email-google, calendar-google, document-google, contacts-google), RequiresScopes annotation
**Exploration:** quick
**Status:** captured

## D3: Single document-microsoft module with two CDI beans (REVISED)

**Choice:** One `document-microsoft` Maven module containing two `@ApplicationScoped` CDI beans: `MicrosoftDrivePlatform` (id = `"microsoft-onedrive"`, user-scoped `/me/drive/*`) and `SharePointDrivePlatform` (id = `"microsoft-sharepoint"`, site-scoped `/sites/{id}/drive/*`). Shared drive logic via composed `MicrosoftDriveClient` taking `basePath` constructor parameter (composition over inheritance).
**Alternatives:**
- Separate modules (`document-microsoft` + `document-sharepoint`) + shared abstract base — over-decomposed; two modules for a single-parameter difference. When you need a shared base class to de-duplicate two modules, the two modules should be one.
- Single module with config — muddies identity (`id()` returns what?)
**Rationale:** Google handles personal drives and shared drives in a single `document-google` module. OneDrive and SharePoint use identical Graph API drive operations — only the base path differs. Two CDI beans in one module gives distinct `id()` values and `@RequiresScopes` per bean without module proliferation. Composition (`MicrosoftDriveClient` with basePath parameter) is simpler than inheritance for a single-string difference.
**Trade-offs:** Both beans share a module, so a deployment wanting only SharePoint must include OneDrive code (but not register the bean). Acceptable for the code sharing benefit.
**Sources:** Spec (docs/specs/issue-211-microsoft-graph/README.md §Module Structure), DocumentPlatform SPI, Google precedent (single document-google module), decision review R1-03
**Exploration:** quick
**Status:** revised (from: separate modules)

## D4: Include chat-teams in scope

**Choice:** Include `chat-teams` (ChatPlatform via Graph API) in the epic scope
**Alternatives:**
- Defer to follow-up issue — reduces scope from L to M, but leaves the Microsoft integration story incomplete
**Rationale:** The shared client and credential infrastructure will be built regardless. Adding chat-teams while the Graph API patterns are fresh avoids a second context-loading cost. Completes the Microsoft integration in one pass.
**Trade-offs:** Increases scope. Teams ChatPlatform may require application permissions (not just delegated) for some operations, adding auth complexity.
**Sources:** Issue #211 scope section, gap analysis, ChatPlatform SPI (chat-spi)
**Exploration:** quick
**Status:** captured

## D5: Thin base client + domain-specific clients

**Choice:** `MicrosoftGraphClient` in `microsoft-graph-client` provides low-level HTTP operations (get/post/patch/delete with pagination and error mapping). Each provider module has its own domain client (e.g. `MicrosoftMailClient`, `MicrosoftCalendarClient`) that uses `MicrosoftGraphClient` for HTTP.
**Alternatives:**
- Monolithic shared client — one `MicrosoftGraphClient` with all domain methods. Centralised but grows large, couples all providers to one class, violates single responsibility
- Base client only — platform impls call `MicrosoftGraphClient.get()` directly with raw paths. Minimal abstraction but path construction and response parsing scattered across platform impls
**Rationale:** Each provider module is self-contained with its own domain client that knows its API paths and response shapes. The shared base client handles only cross-cutting concerns: base URL, auth header injection, pagination (`@odata.nextLink`), error mapping, and `@Blocking` annotation compliance. Changes to one domain (e.g. mail-specific pagination quirks) don't ripple to others.
**Trade-offs:** More classes per module (domain client + platform impl + optional mapper). Worth it for isolation.
**Sources:** SlackBotClient pattern (16 methods in one class — works at that scale but Graph covers more domains), DiscordClient pattern, shared-http-client protocol
**Exploration:** quick
**Status:** captured

## D6: Composed DriveClient with basePath parameter (REVISED)

**Choice:** `MicrosoftDriveClient` in `document-microsoft` module — a composed helper class that takes `basePath` as a constructor parameter and implements all drive CRUD operations. Both `MicrosoftDrivePlatform` and `SharePointDrivePlatform` create their own `MicrosoftDriveClient` instance with their respective base path.
**Alternatives:**
- Abstract base class with template method — inheritance for a single-string difference is over-engineered. The two platform beans don't need to be polymorphic with each other, only with `DocumentPlatform`.
- Independent implementations — duplication of identical logic
**Rationale:** Composition over inheritance. `MicrosoftDriveClient("/me/drive")` vs `MicrosoftDriveClient("/sites/{siteId}/drive")` — the difference is a constructor argument, not a class hierarchy.
**Trade-offs:** None significant. If SharePoint-specific operations emerge, `SharePointDrivePlatform` can add methods directly without the composed client.
**Sources:** Decision review R1-06, DocumentPlatform SPI
**Depends on:** D3 (revised — single module)
**Exploration:** quick
**Status:** revised (from: shared abstract base)

## D7: Per-provider OAuth2 scope declarations

**Choice:** Each platform implementation uses `@RequiresScopes(provider = "microsoft", scopes = {...})` to declare only the scopes it needs. `ServiceConnectionProvider` merges scopes at consent time based on which providers are active in the deployment.
**Alternatives:**
- Combined multi-scope — one broad scope set for all Microsoft providers. Simpler token management but over-privileged
**Rationale:** Principle of least privilege. A deployment using only `email-microsoft` requests only `Mail.Read`, `Mail.ReadWrite`. Deployments with all 5 providers get the union. Follows the same pattern as Google providers.
**Trade-offs:** Token may need re-consent if a new provider is added to an existing deployment (scope upgrade). Standard OAuth2 behaviour, not a platform-specific issue.
**Sources:** @RequiresScopes annotation (platform-api), Google provider pattern
**Depends on:** D2 (ServiceConnectionProvider)
**Exploration:** quick
**Status:** captured
