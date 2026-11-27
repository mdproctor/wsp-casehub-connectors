# HANDOFF — casehub-connectors (#375)

## Last Session

Completed #375 (google-oauth-family) on branch `issue-375-google-oauth-family`. Single Google OAuth consent now activates all 4 Google connectors (Calendar, Drive, Contacts, Gmail).

**What was built:**

- **Connector manifests** — created for calendar-google, document-google, contacts-google; simplified email-google. All reference `auth.family: google`; `enrichWithFamily()` fills in OAuth config from the family manifest.
- **OAuthConsentService enhancements** — stores `expires-at` and `granted-scopes` in `ProviderCredentialFacade`; adds `include_granted_scopes=true` for incremental consent families; adds `buildAuthorizationUrl` overload for per-connector scope subsets. `OAuthConfigRegistry` now carries `ConsentStrategy` alongside `OAuthDeclaration`.
- **FamilyConnectionProvider** — `ServiceConnectionProvider` implementation in the graphql module. Reads tokens from `ProviderCredentialFacade`, handles refresh via family manifest's token URL with per-provider synchronization. Registered as `@ApplicationScoped @DefaultBean` — platform's `ServiceConnectionProviderCore` takes priority in full-platform deployments.
- **Family-aware DependencyResolver** — auto-infers OAuth requirements from `auth.family` during construction. `FamilyOAuthOption` added to sealed `ResolutionOption`. `DependencyResolverCreator` resolves `ProviderCredentialFacade` via `Arc.container().select()` for optional injection.
- **Integration test** — verifies family resolution, unmet→satisfied dependency transition, and credential store propagation.

**Design decisions shaped by adversarial review:** Originally planned to bridge tokens into platform's `OAuthTokenStore`, but review surfaced per-actor data model mismatch and dependency direction violation. Revised to connector-side `FamilyConnectionProvider` approach.

Squash review: 14 → 6 commits.

## Follow-Up Issues Filed

- **#380** — CAPTCHA pass-through migration from Neocortex to connectors (S/Med)
- **#381** — Composite `ServiceConnectionProvider` for full-platform deployments (S/Low)
- **#382** — Fix MCP server test CDI unsatisfied dependency (pre-existing) (XS/Low)

## Immediate Next Step

Next queue items: #380, #381, #382 (follow-ups from this session), then whatever was already queued.

## Known Issues

- **MCP server test failure** (pre-existing, #382) — `McpServerStartupTest` fails because `CredentialFormService` requires `StaticCredentialStore` which isn't available in the MCP test CDI context. Predates this branch — confirmed by testing on main.
- **`FamilyManifestLoader.discover()` only handles `file:` protocol** — family manifests in JAR dependencies aren't found. Integration tests construct families programmatically as a workaround. Not a runtime issue (Quarkus classpath resources are file-based during dev mode).

## References

- Design spec: `docs/specs/issue-375-google-oauth-family/2026-11-27-google-oauth-family-design.md`
- Decisions: `work/specs/issue-375-google-oauth-family/decisions.md`
- Decision review: `/Users/mdproctor/reviews/casehub-connectors/issue-375-decision-20261127-150122/`
- Diary: `docs/blog/2026-11-27-mdp02-one-consent-to-rule-them-all.md`
