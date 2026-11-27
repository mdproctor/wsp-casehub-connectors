# TrueLayer PSD2 Consent — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #378 — TrueLayer PSD2 consent flow via manifest
**Issue group:** #378

**Goal:** Integrate TrueLayer's existing PSD2 consent flow into the manifest system with per-user token resolution via a new UserTokenResolver SPI.

**Architecture:** Add `perUser` to FlowDeclaration, create UserTokenResolver CDI SPI, make FamilyConnectionProvider actorId-aware to delegate per-user flows, have TrueLayerConsentService implement the SPI, update the TrueLayer manifest with OAuth endpoints and flows.

**Tech Stack:** Java 21, Quarkus 3.32.2, Jackson YAML, JUnit 5, AssertJ

## Global Constraints

- Java source level 21, run on Java 26 JVM
- Build: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn clean install`
- Use `ide_insert_member` / `ide_replace_member` for structural edits
- Credentials at call time, never on shared clients (credential-config-ownership protocol)
- TrueLayerConsentService internals unchanged — only add SPI implementation

---

## Batch 1: UserTokenResolver SPI and FlowDeclaration per-user flag

### Task 1: Create UserTokenResolver interface and add perUser to FlowDeclaration

**Files:**
- Create: `connectors-api/src/main/java/io/casehub/connectors/lifecycle/UserTokenResolver.java`
- Modify: `connectors-api/src/main/java/io/casehub/connectors/lifecycle/FamilyManifest.java:21-24`
- Test: `connectors-api/src/test/java/io/casehub/connectors/lifecycle/FamilyManifestTest.java`

**Interfaces:**
- Produces: `UserTokenResolver { String providerId(); Optional<String> getAccessToken(String userId); }`
- Produces: `FlowDeclaration.perUser()` — `Boolean`, nullable, defaults to false via `isPerUser()`

- [ ] **Step 1: Write failing test for FlowDeclaration with per-user flag**

```java
@Test
void parses_flow_with_per_user_flag() throws Exception {
    var yaml = """
            family:
              id: truelayer-test
              description: "Test"
              consent: aggregated
              oauth:
                auth-url: "https://auth.example.com"
                token-url: "https://auth.example.com/token"
                redirect-uri: "http://localhost/callback"
              flows:
                delegated:
                  grant: authorization_code
                  scopes:
                    - accounts
                  per-user: true
                app:
                  grant: client_credentials
                  scopes:
                    - "https://api.example.com/.default"
              fields:
                - name: client-id
                  sensitive: false
            """;
    var family = MAPPER.readValue(yaml, FamilyManifest.class);
    var delegated = family.family().flows().get("delegated");
    assertThat(delegated.isPerUser()).isTrue();

    var app = family.family().flows().get("app");
    assertThat(app.isPerUser()).isFalse();
}

@Test
void flow_without_per_user_defaults_to_false() throws Exception {
    var yaml = """
            family:
              id: google-test
              description: "Test"
              consent: incremental
              oauth:
                auth-url: "https://accounts.google.com"
                token-url: "https://oauth2.googleapis.com/token"
                redirect-uri: "http://localhost/callback"
              flows:
                delegated:
                  grant: authorization_code
                  scopes:
                    - calendar.readonly
              fields:
                - name: client-id
                  sensitive: false
            """;
    var family = MAPPER.readValue(yaml, FamilyManifest.class);
    assertThat(family.family().flows().get("delegated").isPerUser()).isFalse();
}
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl connectors-api -Dtest=FamilyManifestTest -Dsurefire.failIfNoSpecifiedTests=false`
Expected: Compilation failure — `isPerUser()` doesn't exist on FlowDeclaration

- [ ] **Step 3: Create UserTokenResolver interface**

Create `connectors-api/src/main/java/io/casehub/connectors/lifecycle/UserTokenResolver.java`:

```java
package io.casehub.connectors.lifecycle;

import java.util.Optional;

public interface UserTokenResolver {
    String providerId();
    Optional<String> getAccessToken(String userId);
}
```

- [ ] **Step 4: Add perUser to FlowDeclaration**

Update FlowDeclaration record (line 21-24) to add `perUser` field and `isPerUser()` accessor:

```java
public record FlowDeclaration(
        GrantType grant,
        List<String> scopes,
        @JsonProperty("token-url") String tokenUrl,
        @JsonProperty("per-user") Boolean perUser) {

    public boolean isPerUser() {
        return perUser != null && perUser;
    }
}
```

- [ ] **Step 5: Fix compilation errors from FlowDeclaration constructor changes**

FlowDeclaration now has 4 parameters. Find all call sites using `ide_find_references` and add `null` for perUser. Known sites:
- `FamilyConnectionProviderTest.microsoftFamilyWithFlows()` — 3 FlowDeclaration constructors
- Any other test constructors

- [ ] **Step 6: Run tests to verify they pass**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl connectors-api -Dtest=FamilyManifestTest -Dsurefire.failIfNoSpecifiedTests=false`
Expected: All tests PASS

- [ ] **Step 7: Build full project to verify no regressions**

Run: `ide_build_project`
Expected: 0 errors

- [ ] **Step 8: Commit**

```bash
git add connectors-api/src/main/java/io/casehub/connectors/lifecycle/UserTokenResolver.java connectors-api/src/main/java/io/casehub/connectors/lifecycle/FamilyManifest.java connectors-api/src/test/java/io/casehub/connectors/lifecycle/FamilyManifestTest.java
git commit -m "feat(#378): UserTokenResolver SPI and per-user flag on FlowDeclaration

Refs #378"
```

## Batch 2: FamilyConnectionProvider actorId-aware routing

### Task 2: FamilyConnectionProvider delegates to UserTokenResolver for per-user flows

**Files:**
- Modify: `graphql/src/main/java/io/casehub/connectors/graphql/FamilyConnectionProvider.java:78-88,91-120`
- Test: `graphql/src/test/java/io/casehub/connectors/graphql/FamilyConnectionProviderTest.java`

**Interfaces:**
- Consumes: `UserTokenResolver` (from Task 1), `FlowDeclaration.isPerUser()` (from Task 1)
- Produces: `FamilyConnectionProvider` constructor accepting `Map<String, UserTokenResolver>`

- [ ] **Step 1: Write failing test for per-user token delegation**

```java
@Test
void per_user_flow_delegates_to_UserTokenResolver() {
    var facade = mutableFacade(Map.of());
    var resolver = new TestUserTokenResolver("truelayer", Map.of("user1", "user1-token"));
    var family = truelayerFamilyWithPerUserFlow();
    var provider = new FamilyConnectionProvider(facade,
            Map.of("truelayer", family), Map.of("truelayer", resolver));

    var token = provider.getAccessToken("user1", "truelayer", "default");
    assertThat(token.accessToken()).isEqualTo("user1-token");
}

@Test
void per_user_flow_throws_when_no_user_token() {
    var facade = mutableFacade(Map.of());
    var resolver = new TestUserTokenResolver("truelayer", Map.of());
    var family = truelayerFamilyWithPerUserFlow();
    var provider = new FamilyConnectionProvider(facade,
            Map.of("truelayer", family), Map.of("truelayer", resolver));

    assertThatThrownBy(() -> provider.getAccessToken("user1", "truelayer", "default"))
            .isInstanceOf(ServiceConnectionException.class)
            .hasMessageContaining("No user token");
}

@Test
void non_per_user_flow_ignores_actorId_uses_facade() {
    var facade = mutableFacade(Map.of(
            "google", Map.of(
                    "access-token", "tenant-token",
                    "expires-at", Instant.now().plusSeconds(3600).toString())));
    var provider = new FamilyConnectionProvider(facade,
            Map.of("google", googleFamily()), Map.of());

    var token = provider.getAccessToken("user1", "google", "default");
    assertThat(token.accessToken()).isEqualTo("tenant-token");
}

@Test
void null_actorId_bypasses_per_user_even_if_flow_is_per_user() {
    var facade = mutableFacade(Map.of(
            "truelayer", Map.of(
                    "access-token", "tenant-fallback",
                    "expires-at", Instant.now().plusSeconds(3600).toString())));
    var resolver = new TestUserTokenResolver("truelayer", Map.of("user1", "user1-token"));
    var family = truelayerFamilyWithPerUserFlow();
    var provider = new FamilyConnectionProvider(facade,
            Map.of("truelayer", family), Map.of("truelayer", resolver));

    var token = provider.getAccessToken(null, "truelayer", "default");
    assertThat(token.accessToken()).isEqualTo("tenant-fallback");
}
```

Helper methods:

```java
private static FamilyManifest truelayerFamilyWithPerUserFlow() {
    var oauth = new FamilyManifest.OAuthDeclaration(
            "https://auth.truelayer.com/connect/auth",
            "https://auth.truelayer.com/connect/token",
            "http://localhost/callback", null, null, null);
    var flows = Map.of(
            "delegated", new FamilyManifest.FlowDeclaration(
                    FamilyManifest.GrantType.AUTHORIZATION_CODE,
                    List.of("accounts", "balance"), null, true),
            "app", new FamilyManifest.FlowDeclaration(
                    FamilyManifest.GrantType.CLIENT_CREDENTIALS,
                    List.of("https://api.truelayer.com/.default"), null, null));
    return new FamilyManifest(new FamilyManifest.FamilyDeclaration(
            "truelayer", "TrueLayer", FamilyManifest.ConsentStrategy.AGGREGATED,
            oauth, flows, List.of()));
}

static class TestUserTokenResolver implements UserTokenResolver {
    private final String provider;
    private final Map<String, String> tokens;

    TestUserTokenResolver(String provider, Map<String, String> tokens) {
        this.provider = provider;
        this.tokens = tokens;
    }

    @Override
    public String providerId() { return provider; }

    @Override
    public Optional<String> getAccessToken(String userId) {
        return Optional.ofNullable(tokens.get(userId));
    }
}
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl graphql -Dtest=FamilyConnectionProviderTest -Dsurefire.failIfNoSpecifiedTests=false`
Expected: Compilation failure — 3-arg constructor doesn't exist

- [ ] **Step 3: Add userTokenResolvers field and 3-arg constructor**

Add field:
```java
private final Map<String, UserTokenResolver> userTokenResolvers;
```

Add 3-arg constructor:
```java
public FamilyConnectionProvider(ProviderCredentialFacade facade,
                                 Map<String, FamilyManifest> families,
                                 Map<String, UserTokenResolver> userTokenResolvers) {
    this.facade = facade;
    this.families = Map.copyOf(families);
    this.userTokenResolvers = userTokenResolvers != null ? Map.copyOf(userTokenResolvers) : Map.of();
}
```

Update existing 2-arg constructor to delegate:
```java
public FamilyConnectionProvider(ProviderCredentialFacade facade,
                                 Map<String, FamilyManifest> families) {
    this(facade, families, Map.of());
}
```

Update `@Inject` constructor to discover UserTokenResolver beans:
```java
@Inject
public FamilyConnectionProvider(Instance<ProviderCredentialFacade> facadeInstance,
                                 Instance<UserTokenResolver> resolverInstances) {
    this(facadeInstance.isResolvable() ? facadeInstance.get() : null,
         FamilyManifestLoader.discover(),
         buildResolverMap(resolverInstances));
}

private static Map<String, UserTokenResolver> buildResolverMap(Instance<UserTokenResolver> instances) {
    var map = new java.util.HashMap<String, UserTokenResolver>();
    if (instances.isResolvable()) {
        for (var r : instances) {
            map.put(r.providerId(), r);
        }
    }
    return map;
}
```

- [ ] **Step 4: Update getAccessToken to check per-user flows**

Insert per-user check at the top of `getAccessToken`, after `resolveFamily`:

```java
if (actorId != null && resolved.flow() != null && resolved.flow().isPerUser()) {
    var resolver = userTokenResolvers.get(resolved.familyId());
    if (resolver == null) {
        resolver = userTokenResolvers.get(provider);
    }
    if (resolver != null) {
        var token = resolver.getAccessToken(actorId)
                .orElseThrow(() -> new ServiceConnectionException(
                        "No user token for " + provider + " user " + actorId,
                        provider, actorId, Set.of(), Set.of(), Set.of()));
        return new ServiceAccessToken(token, null, Set.of());
    }
}
```

- [ ] **Step 5: Run tests to verify they pass**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl graphql -Dtest=FamilyConnectionProviderTest -Dsurefire.failIfNoSpecifiedTests=false`
Expected: All tests PASS (including existing tests — they use the 2-arg constructor which defaults to empty resolvers)

- [ ] **Step 6: Commit**

```bash
git add graphql/src/main/java/io/casehub/connectors/graphql/FamilyConnectionProvider.java graphql/src/test/java/io/casehub/connectors/graphql/FamilyConnectionProviderTest.java
git commit -m "feat(#378): FamilyConnectionProvider actorId-aware per-user delegation

Refs #378"
```

## Batch 3: TrueLayer manifest and consent service integration

### Task 3: TrueLayerConsentService implements UserTokenResolver

**Files:**
- Modify: `bank-truelayer/src/main/java/io/casehub/connectors/bank/truelayer/TrueLayerConsentService.java`
- Test: `bank-truelayer/src/test/java/io/casehub/connectors/bank/truelayer/TrueLayerConsentServiceTest.java`

**Interfaces:**
- Consumes: `UserTokenResolver` (from Task 1)
- Produces: `TrueLayerConsentService.providerId()` → `"truelayer"`, `TrueLayerConsentService.getAccessToken(userId)` → delegates to `getUserToken(userId)`

- [ ] **Step 1: Write failing test for UserTokenResolver implementation**

```java
@Test
void implements_UserTokenResolver_with_correct_providerId() {
    assertThat(consentService.providerId()).isEqualTo("truelayer");
}

@Test
void getAccessToken_returns_token_via_getUserToken() {
    // Store a consent for the test user
    store.store("user1", new StoredConsent(
            "access-token-1", "refresh-token-1",
            Instant.now().plusSeconds(3600), Instant.now().plusDays(90),
            List.of(ConsentScope.ACCOUNTS), Instant.now()));

    var result = consentService.getAccessToken("user1");
    assertThat(result).isPresent().hasValue("access-token-1");
}

@Test
void getAccessToken_returns_empty_when_no_consent() {
    var result = consentService.getAccessToken("unknown-user");
    assertThat(result).isEmpty();
}
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl bank-truelayer -Dtest=TrueLayerConsentServiceTest -Dsurefire.failIfNoSpecifiedTests=false`
Expected: Compilation failure — `providerId()` doesn't exist

- [ ] **Step 3: Add implements UserTokenResolver to TrueLayerConsentService**

Update class declaration:
```java
public class TrueLayerConsentService implements UserTokenResolver {
```

Add import:
```java
import io.casehub.connectors.lifecycle.UserTokenResolver;
```

Add the two interface methods:
```java
@Override
public String providerId() {
    return "truelayer";
}

@Override
public Optional<String> getAccessToken(String userId) {
    return Optional.ofNullable(getUserToken(userId));
}
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl bank-truelayer -Dtest=TrueLayerConsentServiceTest -Dsurefire.failIfNoSpecifiedTests=false`
Expected: All tests PASS

- [ ] **Step 5: Commit**

```bash
git add bank-truelayer/src/main/java/io/casehub/connectors/bank/truelayer/TrueLayerConsentService.java bank-truelayer/src/test/java/io/casehub/connectors/bank/truelayer/TrueLayerConsentServiceTest.java
git commit -m "feat(#378): TrueLayerConsentService implements UserTokenResolver

Refs #378"
```

### Task 4: Update TrueLayer connector manifest and DependencyResolver

**Files:**
- Modify: `bank-truelayer/src/main/resources/META-INF/connector-manifest.yaml`
- Modify: `connectors-api/src/main/java/io/casehub/connectors/lifecycle/DependencyResolver.java`
- Test: `connectors-api/src/test/java/io/casehub/connectors/lifecycle/DependencyResolverTest.java`

**Interfaces:**
- Consumes: `FlowDeclaration.isPerUser()` (from Task 1)

- [ ] **Step 1: Write failing test for DependencyResolver per-user flow checking**

```java
@Test
void per_user_flow_checks_resolver_availability_not_stored_token() {
    var flows = Map.of(
            "delegated", new FamilyManifest.FlowDeclaration(
                    FamilyManifest.GrantType.AUTHORIZATION_CODE,
                    List.of("accounts"), null, true));
    var family = new FamilyManifest(new FamilyManifest.FamilyDeclaration(
            "truelayer", "TrueLayer", FamilyManifest.ConsentStrategy.AGGREGATED,
            new FamilyManifest.OAuthDeclaration(null, null, null, null, null, null),
            flows, List.of()));
    var manifest = new ConnectorManifest(new ConnectorDeclaration(
            "truelayer", "bank", null,
            new ConnectorManifest.AuthDeclaration("oauth2", null,
                    List.of("delegated"), null, null),
            null, null, null, null, null));

    // No stored token, but resolver IS available
    var facade = new InMemoryCredentialFacade(Map.of(
            "truelayer", Map.of("client-id", "cid", "client-secret", "cs")));
    var resolver = new DependencyResolver(List.of(manifest), facade,
            Map.of("truelayer", family), Set.of("truelayer"));

    var unmet = resolver.unmetRequirements("truelayer", new ConnectorRegistry());
    assertThat(unmet).isEmpty();
}

@Test
void per_user_flow_unmet_when_no_resolver_available() {
    var flows = Map.of(
            "delegated", new FamilyManifest.FlowDeclaration(
                    FamilyManifest.GrantType.AUTHORIZATION_CODE,
                    List.of("accounts"), null, true));
    var family = new FamilyManifest(new FamilyManifest.FamilyDeclaration(
            "truelayer", "TrueLayer", FamilyManifest.ConsentStrategy.AGGREGATED,
            new FamilyManifest.OAuthDeclaration(null, null, null, null, null, null),
            flows, List.of()));
    var manifest = new ConnectorManifest(new ConnectorDeclaration(
            "truelayer", "bank", null,
            new ConnectorManifest.AuthDeclaration("oauth2", null,
                    List.of("delegated"), null, null),
            null, null, null, null, null));

    var facade = new InMemoryCredentialFacade(Map.of());
    var resolver = new DependencyResolver(List.of(manifest), facade,
            Map.of("truelayer", family), Set.of());

    var unmet = resolver.unmetRequirements("truelayer", new ConnectorRegistry());
    assertThat(unmet).hasSize(1);
    assertThat(unmet.getFirst().options().getFirst())
            .isInstanceOf(ResolutionOption.FamilyOAuthOption.class);
}
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl connectors-api -Dtest=DependencyResolverTest -Dsurefire.failIfNoSpecifiedTests=false`
Expected: Compilation failure — 4-arg constructor doesn't exist

- [ ] **Step 3: Add availableUserResolvers to DependencyResolver**

Add field and 4-arg constructor:
```java
private final Set<String> availableUserResolvers;

public DependencyResolver(List<ConnectorManifest> manifests, ProviderCredentialFacade credentialFacade,
                           Map<String, FamilyManifest> families, Set<String> availableUserResolvers) {
    // ... existing init ...
    this.availableUserResolvers = availableUserResolvers != null ? Set.copyOf(availableUserResolvers) : Set.of();
}
```

Update existing 3-arg constructor to delegate with empty set:
```java
public DependencyResolver(List<ConnectorManifest> manifests, ProviderCredentialFacade credentialFacade,
                           Map<String, FamilyManifest> families) {
    this(manifests, credentialFacade, families, Set.of());
}
```

- [ ] **Step 4: Update per-user flow check in unmetRequirements**

In the multi-flow checking block, for `authorization_code` flows with `isPerUser()`:

```java
if (flow.grant() == FamilyManifest.GrantType.AUTHORIZATION_CODE) {
    if (flow.isPerUser()) {
        var resolverKey = providerKey.equals(familyId) ? familyId : providerKey;
        if (!availableUserResolvers.contains(resolverKey) &&
                !availableUserResolvers.contains(familyId)) {
            var members = familyMembers.getOrDefault(familyId, List.of());
            result.add(new UnmetRequirement("oauth2", List.of(flowName),
                    "Per-user consent resolver required for " + familyId + " " + flowName + " flow",
                    List.of(new ResolutionOption.FamilyOAuthOption(familyId, List.copyOf(members)))));
        }
    } else {
        // existing tenant-scoped check
        boolean hasToken = credentialFacade.resolve(providerKey, "access-token").isPresent();
        if (!hasToken) {
            var members = familyMembers.getOrDefault(familyId, List.of());
            result.add(new UnmetRequirement("oauth2", List.of(flowName),
                    "OAuth consent required for " + familyId + " " + flowName + " flow",
                    List.of(new ResolutionOption.FamilyOAuthOption(familyId, List.copyOf(members)))));
        }
    }
}
```

- [ ] **Step 5: Run tests to verify they pass**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl connectors-api -Dtest=DependencyResolverTest -Dsurefire.failIfNoSpecifiedTests=false`
Expected: All tests PASS

- [ ] **Step 6: Update TrueLayer connector manifest**

Replace `bank-truelayer/src/main/resources/META-INF/connector-manifest.yaml`:

```yaml
connector:
  id: truelayer
  platform: bank
  description: "TrueLayer — UK Open Banking (AISP + PISP) via PSD2 consent"
  auth:
    type: oauth2
    oauth:
      auth-url: "{auth-url}/connect/auth"
      token-url: "{auth-url}/connect/token"
      redirect-uri: "{callback-base-url}/auth/truelayer/callback"
    flows:
      delegated:
        grant: authorization_code
        scopes:
          - accounts
          - balance
          - transactions
        per-user: true
      app:
        grant: client_credentials
    fields:
      - name: client-id
        sensitive: false
        description: "TrueLayer client ID"
      - name: client-secret
        description: "TrueLayer client secret"
      - name: auth-url
        required: false
        sensitive: false
        default: "https://auth.truelayer.com"
        description: "TrueLayer auth base URL"
      - name: base-url
        required: false
        sensitive: false
        default: "https://api.truelayer.com"
        description: "TrueLayer API base URL"
  provides:
    - capability: bank
      sub:
        - account-information
        - payment-initiation
  requires:
    - capability: email
      sub: [receive]
      reason: "PSD2 consent flow sends verification emails to the account holder"
```

- [ ] **Step 7: Run full build to verify no regressions**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn clean install -pl '!mcp-server-connectors'`
Expected: BUILD SUCCESS

- [ ] **Step 8: Commit**

```bash
git add connectors-api/src/main/java/io/casehub/connectors/lifecycle/DependencyResolver.java connectors-api/src/test/java/io/casehub/connectors/lifecycle/DependencyResolverTest.java bank-truelayer/src/main/resources/META-INF/connector-manifest.yaml
git commit -m "feat(#378): DependencyResolver per-user awareness + TrueLayer manifest update

Refs #378"
```

## References

- [2026-11-27-truelayer-psd2-consent-design.md] — design spec this plan implements
- `connectors-api/.../FamilyManifest.java:21` — FlowDeclaration record
- `graphql/.../FamilyConnectionProvider.java:91` — getAccessToken (actorId ignored)
- `bank-truelayer/.../TrueLayerConsentService.java` — existing PSD2 consent lifecycle
- `bank-truelayer/.../ConsentTokenStore.java` — per-user token storage interface
- `bank-truelayer/.../connector-manifest.yaml` — existing manifest without OAuth block
- `connectors-api/.../DependencyResolver.java` — multi-flow checking from #376
- GitHub #378 — TrueLayer PSD2 consent flow
- GitHub #376 — Microsoft OAuth family (multi-flow pattern)
