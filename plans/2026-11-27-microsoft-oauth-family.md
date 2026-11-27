# Microsoft OAuth Family — Multi-Flow Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #376 — Microsoft OAuth family manifest
**Issue group:** #376

**Goal:** Extend the family manifest model to support multiple named OAuth flows per family, enabling Microsoft's delegated + application + bot token model.

**Architecture:** Add `FlowDeclaration` and `GrantType` to `FamilyManifest`, extend `FamilyConnectionProvider` with convention-based provider-string-to-flow resolution and `client_credentials` grant support, update `DependencyResolver` for per-flow requirement checking, create missing Microsoft module manifests.

**Tech Stack:** Java 21, Quarkus 3.32.2, Jackson YAML, JUnit 5, AssertJ

## Global Constraints

- Java source level 21, run on Java 26 JVM
- Build: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn clean install`
- All records use compact canonical constructors where validation is needed
- Jackson YAML with `FAIL_ON_UNKNOWN_PROPERTIES=false` and `ACCEPT_CASE_INSENSITIVE_ENUMS=true`
- Credentials at call time, never on shared clients (credential-config-ownership protocol)
- Use `ide_insert_member` / `ide_replace_member` for structural edits

---

## Batch 1: Family manifest multi-flow model

### Task 1: Add FlowDeclaration and GrantType to FamilyManifest

**Files:**
- Modify: `connectors-api/src/main/java/io/casehub/connectors/lifecycle/FamilyManifest.java:6-24`
- Test: `connectors-api/src/test/java/io/casehub/connectors/lifecycle/FamilyManifestTest.java`

**Interfaces:**
- Produces: `FamilyManifest.FlowDeclaration(GrantType grant, List<String> scopes, String tokenUrl)`, `FamilyManifest.GrantType { AUTHORIZATION_CODE, CLIENT_CREDENTIALS }`
- Produces: `FamilyDeclaration.flows()` — returns `Map<String, FlowDeclaration>` (nullable for backward compat)

- [ ] **Step 1: Write failing test for multi-flow family manifest parsing**

```java
@Test
void parses_family_manifest_with_named_flows() throws Exception {
    var yaml = """
            family:
              id: microsoft
              description: "Microsoft 365"
              consent: aggregated
              oauth:
                auth-url: "https://login.microsoftonline.com/{tenant-id}/oauth2/v2/authorize"
                token-url: "https://login.microsoftonline.com/{tenant-id}/oauth2/v2/token"
                redirect-uri: "{callback-base-url}/oauth/callback/microsoft"
              flows:
                delegated:
                  grant: authorization_code
                  scopes:
                    - "Calendars.ReadWrite"
                    - "Mail.Read"
                application:
                  grant: client_credentials
                  scopes:
                    - "https://graph.microsoft.com/.default"
                bot:
                  grant: client_credentials
                  scopes:
                    - "https://api.botframework.com/.default"
                  token-url: "https://login.microsoftonline.com/botframework.com/oauth2/v2.0/token"
              fields:
                - name: tenant-id
                  sensitive: false
                - name: client-id
                  sensitive: false
                - name: client-secret
            """;
    var family = MAPPER.readValue(yaml, FamilyManifest.class);
    assertThat(family.family().id()).isEqualTo("microsoft");
    assertThat(family.family().flows()).hasSize(3);

    var delegated = family.family().flows().get("delegated");
    assertThat(delegated.grant()).isEqualTo(FamilyManifest.GrantType.AUTHORIZATION_CODE);
    assertThat(delegated.scopes()).containsExactly("Calendars.ReadWrite", "Mail.Read");
    assertThat(delegated.tokenUrl()).isNull();

    var app = family.family().flows().get("application");
    assertThat(app.grant()).isEqualTo(FamilyManifest.GrantType.CLIENT_CREDENTIALS);
    assertThat(app.scopes()).containsExactly("https://graph.microsoft.com/.default");

    var bot = family.family().flows().get("bot");
    assertThat(bot.grant()).isEqualTo(FamilyManifest.GrantType.CLIENT_CREDENTIALS);
    assertThat(bot.tokenUrl()).isEqualTo("https://login.microsoftonline.com/botframework.com/oauth2/v2.0/token");
}
```

- [ ] **Step 2: Write failing test for backward compat (no flows)**

```java
@Test
void family_manifest_without_flows_has_null_flows() throws Exception {
    var yaml = """
            family:
              id: google
              description: "Google Workspace"
              consent: incremental
              oauth:
                auth-url: "https://accounts.google.com/o/oauth2/v2/auth"
                token-url: "https://oauth2.googleapis.com/token"
                redirect-uri: "{callback-base-url}/oauth/callback/google"
                scopes:
                  - "https://www.googleapis.com/auth/calendar.readonly"
                access-type: offline
                prompt: consent
              fields:
                - name: client-id
                  sensitive: false
                - name: client-secret
            """;
    var family = MAPPER.readValue(yaml, FamilyManifest.class);
    assertThat(family.family().flows()).isNull();
    assertThat(family.family().oauth().scopes()).containsExactly(
            "https://www.googleapis.com/auth/calendar.readonly");
}
```

- [ ] **Step 3: Run tests to verify they fail**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl connectors-api -Dtest=FamilyManifestTest -Dsurefire.failIfNoSpecifiedTests=false`
Expected: Compilation failure — `FlowDeclaration` and `GrantType` don't exist yet

- [ ] **Step 4: Add FlowDeclaration, GrantType, and flows field to FamilyManifest**

Use `ide_insert_member` to add after `ConsentStrategy` enum (line 15):

```java
public enum GrantType { AUTHORIZATION_CODE, CLIENT_CREDENTIALS }

public record FlowDeclaration(
        GrantType grant,
        List<String> scopes,
        @JsonProperty("token-url") String tokenUrl) {}
```

Use `ide_replace_member` to update `FamilyDeclaration` record (line 8-13) adding `flows` parameter:

```java
public record FamilyDeclaration(
        String id,
        String description,
        ConsentStrategy consent,
        OAuthDeclaration oauth,
        Map<String, FlowDeclaration> flows,
        List<ConnectorManifest.FieldDeclaration> fields) {}
```

Add `import java.util.Map;` to imports.

- [ ] **Step 5: Run tests to verify they pass**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl connectors-api -Dtest=FamilyManifestTest -Dsurefire.failIfNoSpecifiedTests=false`
Expected: All tests PASS (including existing tests — `flows` is null for them, Jackson ignores unknown=false handles missing field)

- [ ] **Step 6: Commit**

```bash
git add connectors-api/src/main/java/io/casehub/connectors/lifecycle/FamilyManifest.java connectors-api/src/test/java/io/casehub/connectors/lifecycle/FamilyManifestTest.java
git commit -m "feat(#376): add FlowDeclaration and GrantType to FamilyManifest

Refs #376"
```

### Task 2: Update microsoft.yaml with named flows

**Files:**
- Modify: `connectors-api/src/main/resources/META-INF/families/microsoft.yaml`
- Test: `connectors-api/src/test/java/io/casehub/connectors/lifecycle/RealFamilyManifestTest.java`

**Interfaces:**
- Consumes: `FamilyManifest.FlowDeclaration`, `FamilyManifest.GrantType` (from Task 1)

- [ ] **Step 1: Write failing test for real microsoft.yaml with flows**

```java
@Test
void real_microsoft_yaml_has_three_flows() {
    var families = FamilyManifestLoader.discover();
    var microsoft = families.get("microsoft");
    assertThat(microsoft).isNotNull();
    assertThat(microsoft.family().flows()).hasSize(3);
    assertThat(microsoft.family().flows()).containsKeys("delegated", "application", "bot");

    var delegated = microsoft.family().flows().get("delegated");
    assertThat(delegated.grant()).isEqualTo(FamilyManifest.GrantType.AUTHORIZATION_CODE);
    assertThat(delegated.scopes()).contains("Calendars.ReadWrite", "Mail.Read", "Mail.Send");

    var app = microsoft.family().flows().get("application");
    assertThat(app.grant()).isEqualTo(FamilyManifest.GrantType.CLIENT_CREDENTIALS);
    assertThat(app.scopes()).containsExactly("https://graph.microsoft.com/.default");

    var bot = microsoft.family().flows().get("bot");
    assertThat(bot.grant()).isEqualTo(FamilyManifest.GrantType.CLIENT_CREDENTIALS);
    assertThat(bot.tokenUrl()).contains("botframework.com");
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl connectors-api -Dtest=RealFamilyManifestTest -Dsurefire.failIfNoSpecifiedTests=false`
Expected: FAIL — `flows()` is null in current microsoft.yaml

- [ ] **Step 3: Update microsoft.yaml**

Replace `connectors-api/src/main/resources/META-INF/families/microsoft.yaml` with:

```yaml
family:
  id: microsoft
  description: "Microsoft 365 — shared Azure AD app registration"
  consent: aggregated
  oauth:
    auth-url: "https://login.microsoftonline.com/{tenant-id}/oauth2/v2/authorize"
    token-url: "https://login.microsoftonline.com/{tenant-id}/oauth2/v2/token"
    redirect-uri: "{callback-base-url}/oauth/callback/microsoft"
  flows:
    delegated:
      grant: authorization_code
      scopes:
        - "Calendars.ReadWrite"
        - "Mail.Read"
        - "Mail.Send"
        - "Files.ReadWrite.All"
        - "Sites.ReadWrite.All"
        - "User.Read"
    application:
      grant: client_credentials
      scopes:
        - "https://graph.microsoft.com/.default"
    bot:
      grant: client_credentials
      scopes:
        - "https://api.botframework.com/.default"
      token-url: "https://login.microsoftonline.com/botframework.com/oauth2/v2.0/token"
  fields:
    - name: tenant-id
      sensitive: false
      description: "Azure AD tenant ID"
      help: "Azure Portal → Azure Active Directory → Overview → Tenant ID"
    - name: client-id
      sensitive: false
      description: "Azure AD application (client) ID"
      help: "Azure Portal → App registrations → your app → Overview → Application (client) ID"
    - name: client-secret
      description: "Azure AD client secret"
      help: "Azure Portal → App registrations → your app → Certificates & secrets → New client secret"
```

- [ ] **Step 4: Run test to verify it passes**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl connectors-api -Dtest=RealFamilyManifestTest -Dsurefire.failIfNoSpecifiedTests=false`
Expected: PASS

- [ ] **Step 5: Verify google.yaml backward compat still works**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl connectors-api -Dtest=FamilyManifestLoaderTest -Dsurefire.failIfNoSpecifiedTests=false`
Expected: All existing tests PASS

- [ ] **Step 6: Commit**

```bash
git add connectors-api/src/main/resources/META-INF/families/microsoft.yaml connectors-api/src/test/java/io/casehub/connectors/lifecycle/RealFamilyManifestTest.java
git commit -m "feat(#376): microsoft.yaml with delegated, application, bot flows

Refs #376"
```

### Task 3: Add flows field to ConnectorManifest.AuthDeclaration and create Microsoft module manifests

**Files:**
- Modify: `connectors-api/src/main/java/io/casehub/connectors/lifecycle/ConnectorManifest.java:25-29`
- Create: `calendar-microsoft/src/main/resources/META-INF/connector-manifest.yaml`
- Create: `document-microsoft/src/main/resources/META-INF/connector-manifest.yaml`
- Create: `chat-teams/src/main/resources/META-INF/connector-manifest.yaml`
- Modify: `email-microsoft/src/main/resources/META-INF/connector-manifest.yaml`
- Test: `connectors-api/src/test/java/io/casehub/connectors/lifecycle/FamilyManifestTest.java`

**Interfaces:**
- Produces: `AuthDeclaration.flows()` — returns `List<String>` (nullable for backward compat)
- Consumes: `FamilyManifest.FlowDeclaration` (from Task 1)

- [ ] **Step 1: Write failing test for AuthDeclaration with flows**

Add to `FamilyManifestTest.java`:

```java
@Test
void auth_declaration_parses_flows_list() throws Exception {
    var yaml = """
            connector:
              id: chat-teams
              platform: chat
              auth:
                type: oauth2
                family: microsoft
                flows:
                  - delegated
                  - application
                  - bot
              provides:
                - capability: chat
            """;
    var manifest = MAPPER.readValue(yaml, ConnectorManifest.class);
    assertThat(manifest.connector().auth().flows())
            .containsExactly("delegated", "application", "bot");
    assertThat(manifest.connector().auth().family()).isEqualTo("microsoft");
}

@Test
void auth_declaration_without_flows_has_null_flows() throws Exception {
    var yaml = """
            connector:
              id: calendar-google
              platform: calendar
              auth:
                type: oauth2
                family: google
              provides:
                - capability: calendar
            """;
    var manifest = MAPPER.readValue(yaml, ConnectorManifest.class);
    assertThat(manifest.connector().auth().flows()).isNull();
}
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl connectors-api -Dtest=FamilyManifestTest#auth_declaration_parses_flows_list -Dsurefire.failIfNoSpecifiedTests=false`
Expected: Compilation failure — `flows()` doesn't exist on `AuthDeclaration`

- [ ] **Step 3: Add flows to AuthDeclaration**

Use `ide_replace_member` to update `AuthDeclaration` record:

```java
public record AuthDeclaration(
        String type,
        String family,
        List<String> flows,
        FamilyManifest.OAuthDeclaration oauth,
        List<FieldDeclaration> fields) {}
```

- [ ] **Step 4: Fix compilation errors from AuthDeclaration constructor changes**

The existing code constructs `AuthDeclaration` with 4 args — now needs 5. Find all call sites using `ide_find_references` on the `AuthDeclaration` constructor and update each. Known call sites:

- `ConnectorManifestLoader.enrichWithFamily()` line 95 — add `oldAuth.flows()` as third arg
- `FamilyManifestTest.credentialKey_returns_family_when_set()` line 139 — add `null` as third arg
- `FamilyManifestTest.credentialKey_returns_id_when_no_family()` line 147 — add `null` as third arg

- [ ] **Step 5: Run tests to verify they pass**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl connectors-api -Dtest=FamilyManifestTest -Dsurefire.failIfNoSpecifiedTests=false`
Expected: All tests PASS

- [ ] **Step 6: Create calendar-microsoft manifest**

Create `calendar-microsoft/src/main/resources/META-INF/connector-manifest.yaml`:

```yaml
connector:
  id: calendar-microsoft
  platform: calendar
  description: "Microsoft Graph Calendar (Outlook) — delegated access"
  auth:
    type: oauth2
    family: microsoft
    flows:
      - delegated
  provides:
    - capability: calendar
      sub:
        - list-calendars
        - list-events
        - get-event
        - create-event
        - update-event
        - delete-event
        - sync-events
```

- [ ] **Step 7: Create document-microsoft manifest**

Create `document-microsoft/src/main/resources/META-INF/connector-manifest.yaml`:

```yaml
connector:
  id: document-microsoft
  platform: document
  description: "Microsoft Graph OneDrive/SharePoint — delegated access"
  auth:
    type: oauth2
    family: microsoft
    flows:
      - delegated
  produces:
    - type: document
      qualifier: microsoft-onedrive
    - type: document
      qualifier: microsoft-sharepoint
  provides:
    - capability: document
      sub:
        - file-operations
        - folder-operations
        - search-operations
        - sharing-operations
```

- [ ] **Step 8: Create chat-teams manifest**

Create `chat-teams/src/main/resources/META-INF/connector-manifest.yaml`:

```yaml
connector:
  id: chat-teams
  platform: chat
  description: "Microsoft Teams — delegated + application + Bot Framework"
  auth:
    type: oauth2
    family: microsoft
    flows:
      - delegated
      - application
      - bot
  provides:
    - capability: chat
      sub:
        - messaging
        - threading
        - discovery
        - presence
        - reactions
        - members
        - channel-management
        - member-management
        - message-history
```

- [ ] **Step 9: Update email-microsoft manifest (remove redundant fields, add flows)**

Replace `email-microsoft/src/main/resources/META-INF/connector-manifest.yaml`:

```yaml
connector:
  id: email-microsoft
  platform: email
  description: "Microsoft Graph Outlook Mail — delegated access"
  auth:
    type: oauth2
    family: microsoft
    flows:
      - delegated
  provides:
    - capability: email
      sub:
        - receive
        - search
```

- [ ] **Step 10: Commit**

```bash
git add connectors-api/src/main/java/io/casehub/connectors/lifecycle/ConnectorManifest.java connectors-api/src/main/java/io/casehub/connectors/lifecycle/ConnectorManifestLoader.java connectors-api/src/test/java/io/casehub/connectors/lifecycle/FamilyManifestTest.java calendar-microsoft/src/main/resources/META-INF/connector-manifest.yaml document-microsoft/src/main/resources/META-INF/connector-manifest.yaml chat-teams/src/main/resources/META-INF/connector-manifest.yaml email-microsoft/src/main/resources/META-INF/connector-manifest.yaml
git commit -m "feat(#376): flows on AuthDeclaration + Microsoft module manifests

Refs #376"
```

## Batch 2: FamilyConnectionProvider multi-flow resolution

### Task 4: Provider string → family:flow resolution in FamilyConnectionProvider

**Files:**
- Modify: `graphql/src/main/java/io/casehub/connectors/graphql/FamilyConnectionProvider.java`
- Test: `graphql/src/test/java/io/casehub/connectors/graphql/FamilyConnectionProviderTest.java`

**Interfaces:**
- Consumes: `FamilyManifest.FlowDeclaration`, `FamilyManifest.GrantType` (from Task 1)
- Produces: `FamilyConnectionProvider.ResolvedFlow(FamilyManifest family, FlowDeclaration flow, String providerKey)` — internal record for resolved family+flow

- [ ] **Step 1: Write failing test for provider string resolution**

Add to `FamilyConnectionProviderTest.java`:

```java
@Test
void resolves_microsoft_to_delegated_flow() {
    var provider = buildProviderWithMicrosoftFlows();
    storeMicrosoftCredentials(provider);
    storeMicrosoftAccessToken(provider, "microsoft");

    var token = provider.getAccessToken(null, "microsoft", "default");
    assertThat(token.accessToken()).isEqualTo("delegated-token");
}

@Test
void resolves_microsoft_app_to_application_flow() {
    var provider = buildProviderWithMicrosoftFlows();
    storeMicrosoftCredentials(provider);
    storeMicrosoftAccessToken(provider, "microsoft-app");

    var token = provider.getAccessToken("__application__", "microsoft-app", "default");
    assertThat(token.accessToken()).isEqualTo("app-token");
}

@Test
void resolves_microsoft_bot_to_bot_flow() {
    var provider = buildProviderWithMicrosoftFlows();
    storeMicrosoftCredentials(provider);
    storeMicrosoftAccessToken(provider, "microsoft-bot");

    var token = provider.getAccessToken("__application__", "microsoft-bot", "default");
    assertThat(token.accessToken()).isEqualTo("bot-token");
}

@Test
void unknown_provider_throws_ServiceConnectionException() {
    var provider = buildProviderWithMicrosoftFlows();
    assertThatThrownBy(() -> provider.getAccessToken(null, "unknown-provider", "default"))
            .isInstanceOf(ServiceConnectionException.class)
            .hasMessageContaining("No family for provider");
}
```

Helper methods:

```java
private FamilyConnectionProvider buildProviderWithMicrosoftFlows() {
    var flows = Map.of(
            "delegated", new FamilyManifest.FlowDeclaration(
                    FamilyManifest.GrantType.AUTHORIZATION_CODE,
                    List.of("Calendars.ReadWrite"), null),
            "application", new FamilyManifest.FlowDeclaration(
                    FamilyManifest.GrantType.CLIENT_CREDENTIALS,
                    List.of("https://graph.microsoft.com/.default"), null),
            "bot", new FamilyManifest.FlowDeclaration(
                    FamilyManifest.GrantType.CLIENT_CREDENTIALS,
                    List.of("https://api.botframework.com/.default"),
                    "https://login.microsoftonline.com/botframework.com/oauth2/v2.0/token")
    );
    var family = new FamilyManifest(new FamilyManifest.FamilyDeclaration(
            "microsoft", "Microsoft 365", FamilyManifest.ConsentStrategy.AGGREGATED,
            new FamilyManifest.OAuthDeclaration(
                    "https://login.microsoftonline.com/test-tenant/oauth2/v2/authorize",
                    "https://login.microsoftonline.com/test-tenant/oauth2/v2/token",
                    "http://localhost/callback", null, null, null),
            flows, List.of()));
    return new FamilyConnectionProvider(facade, Map.of("microsoft", family));
}

private void storeMicrosoftCredentials(FamilyConnectionProvider provider) {
    facade.store("microsoft", "client-id", "test-client-id");
    facade.store("microsoft", "client-secret", "test-secret");
}

private void storeMicrosoftAccessToken(FamilyConnectionProvider provider, String providerKey) {
    facade.store(providerKey, "access-token",
            providerKey.equals("microsoft") ? "delegated-token" :
            providerKey.equals("microsoft-app") ? "app-token" : "bot-token");
    facade.store(providerKey, "expires-at", Instant.now().plusSeconds(3600).toString());
}
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl graphql -Dtest=FamilyConnectionProviderTest -Dsurefire.failIfNoSpecifiedTests=false`
Expected: FAIL — `"microsoft-app"` throws `ServiceConnectionException("No family for provider: microsoft-app")`

- [ ] **Step 3: Implement provider string resolution**

Add a `resolveFamily` method to `FamilyConnectionProvider` and update `getAccessToken` to use it:

```java
record ResolvedFlow(FamilyManifest family, FamilyManifest.FlowDeclaration flow, String familyId) {}

private ResolvedFlow resolveFamily(String provider) {
    // Exact match — primary flow (backward compat)
    var family = families.get(provider);
    if (family != null) {
        if (family.family().flows() != null && !family.family().flows().isEmpty()) {
            var primaryFlow = family.family().flows().values().iterator().next();
            return new ResolvedFlow(family, primaryFlow, provider);
        }
        return new ResolvedFlow(family, null, provider);
    }

    // Convention: find longest family ID prefix, split on '-'
    String bestFamilyId = null;
    FamilyManifest bestFamily = null;
    for (var entry : families.entrySet()) {
        var fid = entry.getKey();
        if (provider.startsWith(fid + "-") &&
                (bestFamilyId == null || fid.length() > bestFamilyId.length())) {
            bestFamilyId = fid;
            bestFamily = entry.getValue();
        }
    }

    if (bestFamily != null && bestFamily.family().flows() != null) {
        var suffix = provider.substring(bestFamilyId.length() + 1);
        var flow = bestFamily.family().flows().get(suffix);
        if (flow != null) {
            return new ResolvedFlow(bestFamily, flow, bestFamilyId);
        }
    }

    throw new ServiceConnectionException("No family for provider: " + provider,
            provider, null, Set.of(), Set.of(), Set.of());
}
```

Update `getAccessToken()` to use `resolveFamily()` instead of `families.get(provider)`:

```java
@Override
public ServiceAccessToken getAccessToken(String actorId, String provider, String tenancyId) {
    var resolved = resolveFamily(provider);
    if (facade == null) {
        throw new ServiceConnectionException("No credential store available",
                provider, actorId, Set.of(), Set.of(), Set.of());
    }
    var accessToken = facade.resolve(provider, "access-token")
            .orElseThrow(() -> new ServiceConnectionException(
                    "No access token for provider: " + provider,
                    provider, actorId, Set.of(), Set.of(), Set.of()));
    // ... rest unchanged (expiry check, refresh)
}
```

Also update `getConnection()` and `listConnections()` to handle suffixed provider strings.

- [ ] **Step 4: Run tests to verify they pass**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl graphql -Dtest=FamilyConnectionProviderTest -Dsurefire.failIfNoSpecifiedTests=false`
Expected: All tests PASS

- [ ] **Step 5: Commit**

```bash
git add graphql/src/main/java/io/casehub/connectors/graphql/FamilyConnectionProvider.java graphql/src/test/java/io/casehub/connectors/graphql/FamilyConnectionProviderTest.java
git commit -m "feat(#376): convention-based provider string → family:flow resolution

Refs #376"
```

### Task 5: client_credentials grant support in FamilyConnectionProvider

**Files:**
- Modify: `graphql/src/main/java/io/casehub/connectors/graphql/FamilyConnectionProvider.java`
- Test: `graphql/src/test/java/io/casehub/connectors/graphql/FamilyConnectionProviderTest.java`

**Interfaces:**
- Consumes: `ResolvedFlow` (from Task 4), `FamilyManifest.GrantType` (from Task 1)
- Produces: `acquireApplicationToken(FamilyManifest, FlowDeclaration, String provider)` — acquires token via client_credentials POST

- [ ] **Step 1: Write failing test for client_credentials token acquisition**

```java
@Test
void acquires_application_token_via_client_credentials_when_missing() {
    var provider = buildProviderWithMicrosoftFlows();
    storeMicrosoftCredentials(provider);
    // No pre-stored token for microsoft-app — should acquire via client_credentials

    // Mock the HTTP POST to return a token
    var testProvider = new TestableFamilyConnectionProvider(facade,
            Map.of("microsoft", buildMicrosoftFamily()));
    testProvider.setTokenResponse("""
            {"access_token": "acquired-app-token", "expires_in": 3600, "token_type": "Bearer"}
            """);

    var token = testProvider.getAccessToken("__application__", "microsoft-app", "default");
    assertThat(token.accessToken()).isEqualTo("acquired-app-token");
    assertThat(testProvider.lastTokenRequestUrl())
            .isEqualTo("https://login.microsoftonline.com/test-tenant/oauth2/v2/token");
    assertThat(testProvider.lastTokenRequestBody())
            .contains("grant_type=client_credentials")
            .contains("scope=" + URLEncoder.encode("https://graph.microsoft.com/.default", StandardCharsets.UTF_8));
}

@Test
void acquires_bot_token_using_flow_specific_token_url() {
    var testProvider = new TestableFamilyConnectionProvider(facade,
            Map.of("microsoft", buildMicrosoftFamily()));
    storeMicrosoftCredentials(testProvider);
    testProvider.setTokenResponse("""
            {"access_token": "bot-token", "expires_in": 3600, "token_type": "Bearer"}
            """);

    var token = testProvider.getAccessToken("__application__", "microsoft-bot", "default");
    assertThat(token.accessToken()).isEqualTo("bot-token");
    assertThat(testProvider.lastTokenRequestUrl())
            .isEqualTo("https://login.microsoftonline.com/botframework.com/oauth2/v2.0/token");
}

@Test
void expired_client_credentials_token_reacquires() {
    var testProvider = new TestableFamilyConnectionProvider(facade,
            Map.of("microsoft", buildMicrosoftFamily()));
    storeMicrosoftCredentials(testProvider);
    // Store expired token
    facade.store("microsoft-app", "access-token", "old-token");
    facade.store("microsoft-app", "expires-at", Instant.now().minusSeconds(60).toString());

    testProvider.setTokenResponse("""
            {"access_token": "fresh-token", "expires_in": 3600, "token_type": "Bearer"}
            """);

    var token = testProvider.getAccessToken("__application__", "microsoft-app", "default");
    assertThat(token.accessToken()).isEqualTo("fresh-token");
}
```

A `TestableFamilyConnectionProvider` subclass that overrides `postTokenRequest()`:

```java
static class TestableFamilyConnectionProvider extends FamilyConnectionProvider {
    private String tokenResponse;
    private String lastUrl;
    private String lastBody;

    TestableFamilyConnectionProvider(ProviderCredentialFacade facade, Map<String, FamilyManifest> families) {
        super(facade, families);
    }

    void setTokenResponse(String response) { this.tokenResponse = response; }
    String lastTokenRequestUrl() { return lastUrl; }
    String lastTokenRequestBody() { return lastBody; }

    @Override
    String postTokenRequest(String tokenUrl, String body) {
        this.lastUrl = tokenUrl;
        this.lastBody = body;
        return tokenResponse;
    }
}
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl graphql -Dtest=FamilyConnectionProviderTest -Dsurefire.failIfNoSpecifiedTests=false`
Expected: FAIL — no client_credentials path exists; throws "No access token" or "No refresh token"

- [ ] **Step 3: Implement client_credentials grant**

Add `acquireApplicationToken` method and update `getAccessToken` to dispatch by grant type:

```java
private ServiceAccessToken acquireApplicationToken(String provider, String actorId,
                                                      ResolvedFlow resolved) {
    var lock = refreshLocks.computeIfAbsent(provider, k -> new Object());
    synchronized (lock) {
        // Double-check: another thread may have acquired
        var currentToken = facade.resolve(provider, "access-token").orElse(null);
        var currentExpiry = facade.resolve(provider, "expires-at")
                .map(Instant::parse).orElse(null);
        if (currentToken != null && currentExpiry != null && currentExpiry.isAfter(Instant.now())) {
            return new ServiceAccessToken(currentToken, currentExpiry, Set.of());
        }

        var tokenUrl = resolved.flow().tokenUrl() != null
                ? resolved.flow().tokenUrl()
                : resolved.family().family().oauth().tokenUrl();
        tokenUrl = resolveTemplate(tokenUrl, resolved.familyId());

        var clientId = facade.resolve(resolved.familyId(), "client-id")
                .orElseThrow(() -> new ServiceConnectionException(
                        "No client-id for family: " + resolved.familyId(),
                        provider, actorId, Set.of(), Set.of(), Set.of()));
        var clientSecret = facade.resolve(resolved.familyId(), "client-secret")
                .orElseThrow(() -> new ServiceConnectionException(
                        "No client-secret for family: " + resolved.familyId(),
                        provider, actorId, Set.of(), Set.of(), Set.of()));

        var scope = String.join(" ", resolved.flow().scopes());
        var body = "grant_type=client_credentials"
                + "&client_id=" + encode(clientId)
                + "&client_secret=" + encode(clientSecret)
                + "&scope=" + encode(scope);

        try {
            var responseBody = postTokenRequest(tokenUrl, body);
            try (var reader = Json.createReader(new StringReader(responseBody))) {
                var json = reader.readObject();
                var accessToken = json.getString("access_token");
                var expiresIn = json.getInt("expires_in", 3600);
                var expiresAt = Instant.now().plusSeconds(expiresIn);

                facade.store(provider, "access-token", accessToken);
                facade.store(provider, "expires-at", expiresAt.toString());

                LOG.info("Application token acquired for provider '" + provider + "'");
                return new ServiceAccessToken(accessToken, expiresAt, Set.of());
            }
        } catch (ServiceConnectionException e) {
            throw e;
        } catch (Exception e) {
            throw new ServiceConnectionException(
                    "Client credentials token acquisition failed: " + e.getMessage(),
                    provider, actorId, Set.of(), Set.of(), Set.of());
        }
    }
}
```

Add `resolveTemplate` method (extract from `OAuthConsentService` pattern or inline):

```java
private String resolveTemplate(String url, String familyId) {
    if (url == null) return null;
    var result = url;
    if (result.contains("{callback-base-url}")) {
        result = result.replace("{callback-base-url}", "http://localhost:8080");
    }
    // Resolve {field-name} from credential facade
    var matcher = java.util.regex.Pattern.compile("\\{([^}]+)}").matcher(result);
    var sb = new StringBuilder();
    while (matcher.find()) {
        var field = matcher.group(1);
        var value = facade.resolve(familyId, field).orElse("{" + field + "}");
        matcher.appendReplacement(sb, java.util.regex.Matcher.quoteReplacement(value));
    }
    matcher.appendTail(sb);
    return sb.toString();
}
```

Update `getAccessToken()` to dispatch:

```java
@Override
public ServiceAccessToken getAccessToken(String actorId, String provider, String tenancyId) {
    var resolved = resolveFamily(provider);
    if (facade == null) {
        throw new ServiceConnectionException("No credential store available",
                provider, actorId, Set.of(), Set.of(), Set.of());
    }

    // For client_credentials flows: auto-acquire if missing or expired
    if (resolved.flow() != null &&
            resolved.flow().grant() == FamilyManifest.GrantType.CLIENT_CREDENTIALS) {
        var accessToken = facade.resolve(provider, "access-token").orElse(null);
        var expiresAt = facade.resolve(provider, "expires-at")
                .map(Instant::parse).orElse(null);
        if (accessToken == null || (expiresAt != null && expiresAt.isBefore(Instant.now()))) {
            return acquireApplicationToken(provider, actorId, resolved);
        }
        return new ServiceAccessToken(accessToken, expiresAt, Set.of());
    }

    // Existing path for authorization_code flows
    var accessToken = facade.resolve(provider, "access-token")
            .orElseThrow(() -> new ServiceConnectionException(
                    "No access token for provider: " + provider,
                    provider, actorId, Set.of(), Set.of(), Set.of()));
    var expiresAt = facade.resolve(provider, "expires-at")
            .map(Instant::parse).orElse(null);
    if (expiresAt != null && expiresAt.isBefore(Instant.now())) {
        return refreshAndReturn(provider, actorId);
    }
    var scopes = parseScopes(facade.resolve(provider, "granted-scopes").orElse(""));
    return new ServiceAccessToken(accessToken, expiresAt, scopes);
}
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl graphql -Dtest=FamilyConnectionProviderTest -Dsurefire.failIfNoSpecifiedTests=false`
Expected: All tests PASS

- [ ] **Step 5: Commit**

```bash
git add graphql/src/main/java/io/casehub/connectors/graphql/FamilyConnectionProvider.java graphql/src/test/java/io/casehub/connectors/graphql/FamilyConnectionProviderTest.java
git commit -m "feat(#376): client_credentials grant support in FamilyConnectionProvider

Refs #376"
```

## Batch 3: DependencyResolver and OAuthConsentService updates

### Task 6: DependencyResolver multi-flow awareness

**Files:**
- Modify: `connectors-api/src/main/java/io/casehub/connectors/lifecycle/DependencyResolver.java`
- Modify: `connectors-api/src/main/java/io/casehub/connectors/lifecycle/ResolutionOption.java`
- Test: `connectors-api/src/test/java/io/casehub/connectors/lifecycle/DependencyResolverTest.java`

**Interfaces:**
- Consumes: `ConnectorManifest.AuthDeclaration.flows()` (from Task 3), `FamilyManifest.flows()` (from Task 1)
- Produces: `ResolutionOption.FamilyClientCredentialsOption(String familyId, String flowName)` — new sealed variant

- [ ] **Step 1: Write failing test for per-flow unmet requirements**

```java
@Test
void generates_separate_unmet_requirements_per_flow() {
    var flows = Map.of(
            "delegated", new FamilyManifest.FlowDeclaration(
                    FamilyManifest.GrantType.AUTHORIZATION_CODE,
                    List.of("User.Read"), null),
            "application", new FamilyManifest.FlowDeclaration(
                    FamilyManifest.GrantType.CLIENT_CREDENTIALS,
                    List.of("https://graph.microsoft.com/.default"), null));
    var family = new FamilyManifest(new FamilyManifest.FamilyDeclaration(
            "microsoft", "Microsoft 365", FamilyManifest.ConsentStrategy.AGGREGATED,
            new FamilyManifest.OAuthDeclaration(null, null, null, null, null, null),
            flows, List.of()));
    var manifest = new ConnectorManifest(new ConnectorManifest.ConnectorDeclaration(
            "chat-teams", "chat", null,
            new ConnectorManifest.AuthDeclaration("oauth2", "microsoft",
                    List.of("delegated", "application"), null, null),
            null, null, null, null, null));
    var resolver = new DependencyResolver(List.of(manifest), facade,
            Map.of("microsoft", family));

    var unmet = resolver.unmetRequirements("chat-teams", new ConnectorRegistry());
    assertThat(unmet).hasSize(2);

    var delegatedReq = unmet.stream()
            .filter(u -> u.sub().contains("delegated")).findFirst().orElseThrow();
    assertThat(delegatedReq.options()).hasSize(1);
    assertThat(delegatedReq.options().get(0)).isInstanceOf(ResolutionOption.FamilyOAuthOption.class);

    var appReq = unmet.stream()
            .filter(u -> u.sub().contains("application")).findFirst().orElseThrow();
    assertThat(appReq.options()).hasSize(1);
    assertThat(appReq.options().get(0)).isInstanceOf(ResolutionOption.FamilyClientCredentialsOption.class);
}

@Test
void no_unmet_requirement_when_delegated_token_exists() {
    // ... same setup but store access-token for "microsoft" in facade
    facade.store("microsoft", "access-token", "test-token");
    // ... should return only the application flow as unmet
}
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl connectors-api -Dtest=DependencyResolverTest -Dsurefire.failIfNoSpecifiedTests=false`
Expected: Compilation failure — `FamilyClientCredentialsOption` doesn't exist, `DependencyResolver` constructor doesn't accept families map

- [ ] **Step 3: Add FamilyClientCredentialsOption to ResolutionOption**

```java
public sealed interface ResolutionOption {
    record ProviderOption(String connectorId, ConnectorStatus status, String setupPlaybook) implements ResolutionOption {}
    record ManualOption(String capability) implements ResolutionOption {}
    record FamilyOAuthOption(String familyId, List<String> memberConnectorIds) implements ResolutionOption {}
    record FamilyClientCredentialsOption(String familyId, String flowName) implements ResolutionOption {}
}
```

- [ ] **Step 4: Update DependencyResolver for multi-flow checking**

Add a constructor overload that accepts families:

```java
public DependencyResolver(List<ConnectorManifest> manifests,
                           ProviderCredentialFacade credentialFacade,
                           Map<String, FamilyManifest> families) {
    this.credentialFacade = credentialFacade;
    this.families = families != null ? Map.copyOf(families) : Map.of();
    // ... existing indexing logic
}
```

Update `unmetRequirements()` OAuth check to handle per-flow:

```java
var auth = manifest.connector().auth();
if (credentialFacade != null && auth != null && "oauth2".equals(auth.type()) && auth.family() != null) {
    var familyId = auth.family();
    var family = families.get(familyId);
    var declaredFlows = auth.flows();

    if (declaredFlows != null && family != null && family.family().flows() != null) {
        for (var flowName : declaredFlows) {
            var flow = family.family().flows().get(flowName);
            if (flow == null) continue;
            var providerKey = flowName.equals(family.family().flows().keySet().iterator().next())
                    ? familyId : familyId + "-" + flowName;
            if (flow.grant() == FamilyManifest.GrantType.AUTHORIZATION_CODE) {
                boolean hasToken = credentialFacade.resolve(providerKey, "access-token").isPresent();
                if (!hasToken) {
                    var members = familyMembers.getOrDefault(familyId, List.of());
                    result.add(new UnmetRequirement("oauth2", List.of(flowName),
                            "OAuth consent required for " + familyId + " " + flowName + " flow",
                            List.of(new ResolutionOption.FamilyOAuthOption(familyId, List.copyOf(members)))));
                }
            } else {
                boolean hasCreds = credentialFacade.resolve(familyId, "client-id").isPresent()
                        && credentialFacade.resolve(familyId, "client-secret").isPresent();
                if (!hasCreds) {
                    result.add(new UnmetRequirement("oauth2", List.of(flowName),
                            "Client credentials required for " + familyId + " " + flowName + " flow",
                            List.of(new ResolutionOption.FamilyClientCredentialsOption(familyId, flowName))));
                }
            }
        }
    } else {
        // Legacy single-flow check (backward compat)
        boolean hasToken = credentialFacade.resolve(familyId, "access-token").isPresent();
        if (!hasToken) {
            var members = familyMembers.getOrDefault(familyId, List.of());
            result.add(new UnmetRequirement("oauth2", List.of(familyId),
                    "OAuth consent required for " + familyId + " family",
                    List.of(new ResolutionOption.FamilyOAuthOption(familyId, List.copyOf(members)))));
        }
    }
}
```

- [ ] **Step 5: Run tests to verify they pass**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl connectors-api -Dtest=DependencyResolverTest -Dsurefire.failIfNoSpecifiedTests=false`
Expected: All tests PASS

- [ ] **Step 6: Commit**

```bash
git add connectors-api/src/main/java/io/casehub/connectors/lifecycle/DependencyResolver.java connectors-api/src/main/java/io/casehub/connectors/lifecycle/ResolutionOption.java connectors-api/src/test/java/io/casehub/connectors/lifecycle/DependencyResolverTest.java
git commit -m "feat(#376): DependencyResolver multi-flow awareness

Refs #376"
```

### Task 7: OAuthConsentService reads scopes from delegated flow

**Files:**
- Modify: `graphql/src/main/java/io/casehub/connectors/graphql/OAuthConsentService.java:107`
- Modify: `connectors-api/src/main/java/io/casehub/connectors/lifecycle/OAuthConfigRegistry.java`
- Modify: `connector-lifecycle/src/main/java/io/casehub/connectors/lifecycle/deployment/OAuthConfigRegistryCreator.java`
- Test: `graphql/src/test/java/io/casehub/connectors/graphql/OAuthConsentServiceTest.java`

**Interfaces:**
- Consumes: `FamilyManifest.flows()` (from Task 1)

- [ ] **Step 1: Write failing test for scope resolution from delegated flow**

```java
@Test
void builds_authorization_url_with_scopes_from_delegated_flow() {
    // Build a family with flows (no top-level scopes)
    var flows = Map.of(
            "delegated", new FamilyManifest.FlowDeclaration(
                    FamilyManifest.GrantType.AUTHORIZATION_CODE,
                    List.of("Mail.Read", "Calendars.ReadWrite"), null));
    var oauth = new FamilyManifest.OAuthDeclaration(
            "https://login.microsoftonline.com/test/oauth2/v2/authorize",
            "https://login.microsoftonline.com/test/oauth2/v2/token",
            "http://localhost:8080/oauth/callback/microsoft",
            null,  // no top-level scopes
            null, null);
    var config = new OAuthConfigRegistry.OAuthConfig(oauth,
            FamilyManifest.ConsentStrategy.AGGREGATED, flows);
    // ... build service and test that the URL contains Mail.Read and Calendars.ReadWrite
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl graphql -Dtest=OAuthConsentServiceTest -Dsurefire.failIfNoSpecifiedTests=false`
Expected: FAIL — `OAuthConfig` doesn't accept flows parameter

- [ ] **Step 3: Update OAuthConfigRegistry.OAuthConfig to include flows**

```java
public record OAuthConfig(
        FamilyManifest.OAuthDeclaration oauth,
        FamilyManifest.ConsentStrategy consent,
        Map<String, FamilyManifest.FlowDeclaration> flows) {

    public List<String> delegatedScopes() {
        if (flows != null && flows.containsKey("delegated")) {
            return flows.get("delegated").scopes();
        }
        return oauth.scopes();
    }
}
```

- [ ] **Step 4: Update OAuthConsentService.buildAuthorizationUrl to use delegatedScopes()**

Change line 107 from:
```java
var scopes = scopeOverride != null ? scopeOverride : new java.util.LinkedHashSet<>(oauth.scopes());
```
to:
```java
var scopes = scopeOverride != null ? scopeOverride : new java.util.LinkedHashSet<>(config.delegatedScopes());
```

- [ ] **Step 5: Update OAuthConfigRegistryCreator to pass flows**

The build-time wiring that creates `OAuthConfig` entries needs to pass the family's `flows` map. Find `OAuthConfigRegistryCreator` and update it to include `family.family().flows()` when constructing `OAuthConfig`.

- [ ] **Step 6: Run tests to verify they pass**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl graphql -Dtest=OAuthConsentServiceTest -Dsurefire.failIfNoSpecifiedTests=false`
Expected: All tests PASS

- [ ] **Step 7: Run full build to verify no regressions**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn clean install`
Expected: BUILD SUCCESS

- [ ] **Step 8: Commit**

```bash
git add graphql/src/main/java/io/casehub/connectors/graphql/OAuthConsentService.java connectors-api/src/main/java/io/casehub/connectors/lifecycle/OAuthConfigRegistry.java connector-lifecycle/src/main/java/io/casehub/connectors/lifecycle/deployment/OAuthConfigRegistryCreator.java graphql/src/test/java/io/casehub/connectors/graphql/OAuthConsentServiceTest.java
git commit -m "feat(#376): OAuthConsentService reads scopes from delegated flow

Refs #376"
```

## References

- [2026-11-27-microsoft-oauth-family-design.md] — design spec this plan implements
- `connectors-api/.../FamilyManifest.java:6` — current single-flow model
- `graphql/.../FamilyConnectionProvider.java:34` — current token resolution (191 lines)
- `graphql/.../OAuthConsentService.java:34` — PKCE consent flow
- `connectors-api/.../DependencyResolver.java:8` — backward chaining
- `connectors-api/.../ConnectorManifest.java:6` — AuthDeclaration without flows
- `connectors-api/.../ConnectorManifestLoader.java:17` — enrichWithFamily()
- `connectors-api/.../ResolutionOption.java:5` — sealed interface (3 variants)
- `connectors-api/.../OAuthConfigRegistry.java:6` — keyed by credential key
- `META-INF/families/microsoft.yaml` — existing family manifest (delegated only)
- `META-INF/families/google.yaml` — reference single-flow family
- GitHub #376 — Microsoft OAuth family manifest
- GitHub #375 — Google family manifest (established pattern)
- GitHub #373 — PersistentCredentialFacade (landed)
