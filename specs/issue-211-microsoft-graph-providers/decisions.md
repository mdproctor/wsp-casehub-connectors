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

## D3: Separate modules for OneDrive and SharePoint

**Choice:** `document-microsoft` (OneDrive, user-scoped) and `document-sharepoint` (SharePoint, site-scoped) as separate Maven modules
**Alternatives:**
- Single module with config — less code but muddies identity (`id()` returns what?), different credential scopes, different discovery model (user drive vs site drive)
**Rationale:** OneDrive is user-scoped (`/me/drive/*`), SharePoint is site-scoped (`/sites/{id}/drive/*`). Different `id()` values, different OAuth2 scopes, different deployment profiles. Clean separation lets deployments include only what they need. Shared code lives in `microsoft-graph-client` via abstract base class (see D6).
**Trade-offs:** Two modules with similar code. Mitigated by shared abstract base in `microsoft-graph-client`.
**Sources:** Spec (docs/specs/issue-211-microsoft-graph/README.md §Module Structure), DocumentPlatform SPI
**Exploration:** quick
**Status:** captured

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

## D6: Shared abstract base for OneDrive/SharePoint drive operations

**Choice:** `AbstractDrivePlatform` in `microsoft-graph-client` implements all `DocumentPlatform` capability sub-interfaces with a `driveBasePath()` template method. `MicrosoftDrivePlatform` and `SharePointPlatform` extend it, providing their path.
**Alternatives:**
- Independent implementations — no coupling but significant duplication of identical drive CRUD logic
- Shared utility class (composition) — `DriveOperations` helper that both delegate to. Similar coupling to base class but loses polymorphic dispatch
**Rationale:** The Graph API drive operations are identical for OneDrive and SharePoint — only the base path differs. A template method pattern captures this precisely. If SharePoint-specific features emerge (e.g. site discovery), the subclass can override individual methods.
**Trade-offs:** Coupling between OneDrive and SharePoint implementations via shared base. If APIs diverge significantly, the base class becomes a liability. Currently unlikely — Microsoft explicitly documents them as the same API.
**Sources:** DocumentPlatform SPI (document-spi), Google DocumentPlatform inner class pattern
**Depends on:** D3 (separate modules)
**Exploration:** quick
**Status:** captured

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
