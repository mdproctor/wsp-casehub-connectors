# HANDOFF — casehub-connectors (#371, #373, #374)

## Last Session

Completed Batch 4 of ConnectorLifecycle Phase 3 and two follow-on issues. Three issues marked done on branch `issue-371-oauth-families-persist-cred`:

- **#371** (epic) — OAuth consent flow infrastructure: `OAuthConsentService` (PKCE, CSRF state, auth URL building, code-for-token exchange), `OAuthCallbackEndpoint` at `/oauth/callback/{family}`, `credentialInput()` OAuth branching (returns OAuth URL for OAuth2 connectors, form URL for the rest), CDI wiring with `@ConfigProperty` for callback base URL.
- **#373** — `PersistentCredentialFacade` with `store()` write side. File-backed at `~/.casehub/credentials.yaml`, `ConcurrentHashMap` for thread-safe runtime access. `DefaultFacadeCreator` updated to create persistent facade instead of `InMemoryCredentialFacade`.
- **#374** — `CredentialFormService.submitCredentials()` now stores to both `StaticCredentialStore` and `ProviderCredentialFacade`, then calls `ConnectorRegistry.updateStatusByCredentialKey()`. Connectors transition UNCONFIGURED → READY at runtime without restart.

Code review caught and fixed: XSS in `OAuthCallbackEndpoint.esc()` (IntelliJ's reformat unescaped HTML entities), and `OAuthConsentService.storeToken()` missing `credentialFacade.store()` call.

Squash review: 14 → 11 commits (3 absorbed — CDI wiring and review fixes folded into their parent commits).

## Immediate Next Step

#377 (github-oauth-manifest) is the next queue item. After that: #375 (google-oauth-family), #376 (microsoft-oauth-family), #378 (truelayer-psd2-consent).

## References

- Diary: `blog/2026-11-27-mdp01-one-consent-to-rule-them-all.md` (updated with consent flow + persistent facade sections)
- Flow diagram: `blog/images/oauth-consent-flow.svg`
- Family manifests: `connectors-api/src/main/resources/META-INF/families/{google,microsoft}.yaml`
