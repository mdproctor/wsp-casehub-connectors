# Service Connections Integration Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #157 — Wire ServiceConnectionProvider into connectors
**Issue group:** #157, #158

**Goal:** Replace all ad-hoc credential resolver interfaces with platform's `ServiceConnectionProvider` and `StaticCredentialStore`, and add `@RequiresScopes` annotations to Google/GitHub consumers.

**Architecture:** Platform shipped `ServiceConnectionProvider` (platform#530) with `getAccessToken(actorId, provider, tenancyId)` for OAuth credentials and `StaticCredentialStore` (platform#548) for API keys/bot tokens. Each connector consumer switches its injection from the per-module resolver to the platform type, builds services using the returned `ServiceAccessToken.accessToken()` via `GoogleCredentials.create(AccessToken)`, and declares its scope requirements via `@RequiresScopes`. All obsolete resolver interfaces and config classes are deleted.

**Tech Stack:** Java 21, Quarkus 3.32.2, platform-api (io.casehub.platform.api.authn), google-auth-library, google-api-services-*

## Global Constraints

- Build: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn clean install`
- All HTTP via `HttpHelper.CLIENT` (shared-http-client protocol)
- SPI identifiers use `id()` not `connectorId()` (spi-id-method-naming protocol)
- Credentials at call time, not on shared clients (credential-config-ownership protocol)
- platform-api dependency: `io.casehub.platform:casehub-platform-api:0.2-SNAPSHOT`
- `@RequiresScopes` is in `io.casehub.platform.api.authn`
- `ServiceConnectionProvider` is in `io.casehub.platform.api.authn`
- `StaticCredentialStore` is in `io.casehub.platform.api.authn`
- `ServiceAccessToken` record: `accessToken()`, `expiresAt()`, `grantedScopes()`
- All platform API keys use 3-part key: `(actorId, provider, tenancyId)` — consumers must thread tenancy

---

## Batch 1: Template migration — GoogleCalendarPlatform

Establishes the pattern all other Google OAuth migrations follow. Calendar-google is the best template because it already has the resolver-based per-user pattern.

### Task 1: Migrate GoogleCalendarPlatform to ServiceConnectionProvider + @RequiresScopes

**Files:**
- Modify: `calendar-google/pom.xml` — add platform-api dependency
- Modify: `calendar-google/src/main/java/io/casehub/connectors/calendar/google/GoogleCalendarPlatform.java` — replace resolver with ServiceConnectionProvider
- Delete: `calendar-google/src/main/java/io/casehub/connectors/calendar/google/GoogleCredentialResolver.java` (use `ide_refactor_safe_delete`)
- Delete: `calendar-google/src/main/java/io/casehub/connectors/calendar/google/GoogleOAuthConfig.java` (use `ide_refactor_safe_delete`)
- Modify: `calendar-google/src/test/java/io/casehub/connectors/calendar/google/GoogleCalendarPlatformTest.java` — update test

**Interfaces:**
- Consumes: `io.casehub.platform.api.authn.ServiceConnectionProvider.getAccessToken(String actorId, String provider, String tenancyId)` → `ServiceAccessToken(accessToken, expiresAt, grantedScopes)`
- Consumes: `io.casehub.platform.api.authn.RequiresScopes` annotation
- Produces: Working GoogleCalendarPlatform that resolves credentials via ServiceConnectionProvider. This is the template pattern for Tasks 2 and 3.

- [ ] **Step 1: Add platform-api dependency to calendar-google/pom.xml**

Add to `<dependencies>`:

```xml
<dependency>
    <groupId>io.casehub.platform</groupId>
    <artifactId>casehub-platform-api</artifactId>
</dependency>
```

Version is managed by parent POM's `<dependencyManagement>`.

- [ ] **Step 2: Write failing test for ServiceConnectionProvider-based credential resolution**

In `GoogleCalendarPlatformTest.java`, add a test that creates `GoogleCalendarPlatform` with a `ServiceConnectionProvider` mock:

```java
@Test
void buildService_usesServiceConnectionProvider() {
    var provider = mock(ServiceConnectionProvider.class);
    var token = new ServiceAccessToken("test-access-token",
        Instant.now().plusSeconds(3600), Set.of("calendar.readonly"));
    when(provider.getAccessToken("user1", "google", "default"))
        .thenReturn(token);

    var platform = new GoogleCalendarPlatform(provider);
    // buildService should use the provider, not throw
    assertDoesNotThrow(() -> platform.buildService("user1", "default"));
}
```

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl calendar-google -Dtest=GoogleCalendarPlatformTest#buildService_usesServiceConnectionProvider`
Expected: FAIL — constructor doesn't accept ServiceConnectionProvider yet.

- [ ] **Step 3: Rewrite GoogleCalendarPlatform to use ServiceConnectionProvider**

Replace the class fields and constructors. The key change: instead of `GoogleCredentialResolver` → `GoogleOAuthConfig(refreshToken, clientId, clientSecret)` → `UserCredentials`, use `ServiceConnectionProvider` → `ServiceAccessToken(accessToken)` → `GoogleCredentials.create(AccessToken)`.

Use `ide_replace_member` to replace the fields and constructors:

```java
@RequiresScopes(provider = "google",
    scopes = {"https://www.googleapis.com/auth/calendar.readonly",
              "https://www.googleapis.com/auth/calendar.events"})
public class GoogleCalendarPlatform implements CalendarPlatform {

    private static final Logger LOG       = Logger.getLogger(GoogleCalendarPlatform.class);
    private static final int    MAX_PAGES = 20;

    private final ServiceConnectionProvider connectionProvider;
    private final NetHttpTransport transport;

    public GoogleCalendarPlatform(ServiceConnectionProvider connectionProvider) {
        this.connectionProvider = connectionProvider;
        try {
            this.transport = GoogleNetHttpTransport.newTrustedTransport();
        } catch (GeneralSecurityException | IOException e) {
            throw new RuntimeException("Failed to initialize HTTP transport", e);
        }
    }

    GoogleCalendarPlatform(Calendar calendarService) {
        this.connectionProvider = null;
        this.transport = null;
        this.calendarService = calendarService;
    }
```

Replace `buildService(String userId)` with `buildService(String actorId, String tenancyId)`:

```java
    Calendar buildService(String actorId, String tenancyId) {
        var token = connectionProvider.getAccessToken(actorId, "google", tenancyId);
        var credentials = GoogleCredentials.create(
            new AccessToken(token.accessToken(), Date.from(token.expiresAt())));
        return new Calendar.Builder(transport, GsonFactory.getDefaultInstance(),
                                    new HttpCredentialsAdapter(credentials))
                       .setApplicationName("casehub-connectors")
                       .build();
    }
```

Remove the `isActive()` method (no longer needed — ServiceConnectionProvider handles availability via DISCONNECTED status) and the raw-credentials constructor.

Update imports: add `io.casehub.platform.api.authn.ServiceConnectionProvider`, `io.casehub.platform.api.authn.RequiresScopes`, `com.google.auth.oauth2.GoogleCredentials`, `com.google.auth.oauth2.AccessToken`. Remove `GoogleCredentialResolver`, `GoogleOAuthConfig`, `UserCredentials`.

Update all internal call sites that called `buildService(userId)` to pass tenancyId. For methods that don't currently take userId (e.g. `listCalendars()`), they already use a pre-built `calendarService` field — these paths remain unchanged (they're the test/static-config path via the `Calendar` constructor).

- [ ] **Step 4: Delete GoogleCredentialResolver and GoogleOAuthConfig from calendar-google**

Use `ide_refactor_safe_delete` on:
- `calendar-google/src/main/java/io/casehub/connectors/calendar/google/GoogleCredentialResolver.java`
- `calendar-google/src/main/java/io/casehub/connectors/calendar/google/GoogleOAuthConfig.java`

- [ ] **Step 5: Update GoogleCalendarPlatformTest**

Update existing tests that use the raw-credentials constructor to use the test-path constructor (`Calendar calendarService`). Update any test that uses `GoogleCredentialResolver` to use `ServiceConnectionProvider` mock instead.

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl calendar-google`
Expected: ALL PASS

- [ ] **Step 6: Verify calendar-google module compiles and tests pass**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn clean install -pl calendar-google`
Expected: BUILD SUCCESS

- [ ] **Step 7: Commit**

```bash
git add calendar-google/
git commit -m "feat(#157,#158): migrate GoogleCalendarPlatform to ServiceConnectionProvider + @RequiresScopes

Delete GoogleCredentialResolver and GoogleOAuthConfig. Use
ServiceConnectionProvider.getAccessToken() with GoogleCredentials.create(AccessToken).
Add @RequiresScopes for calendar.readonly and calendar.events scopes.

Refs #157, #158"
```

---

## Batch 2: Remaining Google + GitHub consumers

Applies the pattern from Batch 1 to the remaining five consumers. Calendar-google established the template; these follow it with minor variations per credential pattern.

### Task 2: Migrate GoogleContactsPlatform, GoogleEmailPlatform, GoogleDocumentPlatform

**Files:**
- Modify: `contacts-google/pom.xml`, `email-google/pom.xml`, `document-google/pom.xml` — add platform-api dependency
- Modify: `contacts-google/src/main/java/io/casehub/connectors/contacts/google/GoogleContactsPlatform.java`
- Modify: `email-google/src/main/java/io/casehub/connectors/email/google/GoogleEmailPlatform.java`
- Modify: `document-google/src/main/java/io/casehub/connectors/document/google/GoogleDocumentPlatform.java`
- Delete: `contacts-google/src/main/java/io/casehub/connectors/contacts/google/GoogleCredentialResolver.java` (use `ide_refactor_safe_delete`)
- Delete: `contacts-google/src/main/java/io/casehub/connectors/contacts/google/GoogleOAuthConfig.java` (use `ide_refactor_safe_delete`)
- Delete: `contacts-google/src/main/java/io/casehub/connectors/contacts/google/ConfigGoogleCredentialResolver.java` (use `ide_refactor_safe_delete`)
- Delete: `email-google/src/main/java/io/casehub/connectors/email/google/EmailGoogleBeans.java` (use `ide_refactor_safe_delete`)
- Delete: `document-google/src/main/java/io/casehub/connectors/document/google/DocumentGoogleBeans.java` (use `ide_refactor_safe_delete`)
- Modify: `contacts-google/src/test/java/io/casehub/connectors/contacts/google/GoogleContactsPlatformTest.java`
- Modify: `email-google/src/test/java/io/casehub/connectors/email/google/GoogleEmailPlatformTest.java`
- Modify: `document-google/src/test/java/io/casehub/connectors/document/google/GoogleDocumentPlatformTest.java`

**Interfaces:**
- Consumes: Same `ServiceConnectionProvider` API as Task 1
- Produces: Three migrated Google consumers, all with `@RequiresScopes`

- [ ] **Step 1: Add platform-api dependency to contacts-google, email-google, document-google pom.xml files**

Same dependency block as Task 1 Step 1 in each module's `pom.xml`.

- [ ] **Step 2: Write failing tests for all three consumers**

For each consumer, add a test verifying `ServiceConnectionProvider` injection. Follow the exact pattern from Task 1 Step 2, substituting the service type (PeopleService, Gmail, Drive).

Run each module's tests to verify failures.

- [ ] **Step 3: Migrate GoogleContactsPlatform**

Same pattern as GoogleCalendarPlatform (Task 1 Step 3). GoogleContactsPlatform already has the resolver-based per-user pattern:

Replace `GoogleCredentialResolver resolver` field → `ServiceConnectionProvider connectionProvider`.

Replace `buildService(String userId)`:

```java
    private PeopleService buildService(String actorId, String tenancyId) {
        var token = connectionProvider.getAccessToken(actorId, "google", tenancyId);
        var credentials = GoogleCredentials.create(
            new AccessToken(token.accessToken(), Date.from(token.expiresAt())));
        return new PeopleService.Builder(transport, GsonFactory.getDefaultInstance(),
                                         new HttpCredentialsAdapter(credentials))
                       .setApplicationName("casehub-connectors")
                       .build();
    }
```

Add `@RequiresScopes`:

```java
@RequiresScopes(provider = "google",
    scopes = {"https://www.googleapis.com/auth/contacts.readonly",
              "https://www.googleapis.com/auth/contacts"})
```

Update user-scoped accessors (`contactRead(String userId)`, etc.) to thread tenancyId — these become `contactRead(String actorId, String tenancyId)` or resolve tenancy from context.

- [ ] **Step 4: Migrate GoogleEmailPlatform**

Bigger change — currently single-tenant (raw credentials from `@ConfigProperty`). Becomes per-user:

Replace `clientId`/`clientSecret`/`refreshToken` fields → `ServiceConnectionProvider connectionProvider`.

Delete `init()` method (no longer builds service at startup).

Add per-request service builder:

```java
    private Gmail buildService(String actorId, String tenancyId) {
        var token = connectionProvider.getAccessToken(actorId, "google", tenancyId);
        var credentials = GoogleCredentials.create(
            new AccessToken(token.accessToken(), Date.from(token.expiresAt())));
        return new Gmail.Builder(GoogleNetHttpTransport.newTrustedTransport(),
                                 GsonFactory.getDefaultInstance(),
                                 new HttpCredentialsAdapter(credentials))
                       .setApplicationName("casehub-connectors")
                       .build();
    }
```

Add `@RequiresScopes`:

```java
@RequiresScopes(provider = "google",
    scopes = {"https://www.googleapis.com/auth/gmail.readonly",
              "https://www.googleapis.com/auth/gmail.modify"})
```

Delete `EmailGoogleBeans.java` (CDI producer no longer needed — `GoogleEmailPlatform` itself becomes `@ApplicationScoped` with constructor injection).

- [ ] **Step 5: Migrate GoogleDocumentPlatform**

Same pattern as GoogleEmailPlatform (Step 4). Replace raw credentials → `ServiceConnectionProvider`. Delete `DocumentGoogleBeans.java`.

Add `@RequiresScopes`:

```java
@RequiresScopes(provider = "google",
    scopes = {"https://www.googleapis.com/auth/drive.readonly",
              "https://www.googleapis.com/auth/drive.file"})
```

- [ ] **Step 6: Delete obsolete files from contacts-google, email-google, document-google**

Use `ide_refactor_safe_delete`:
- `contacts-google/.../GoogleCredentialResolver.java`
- `contacts-google/.../GoogleOAuthConfig.java`
- `contacts-google/.../ConfigGoogleCredentialResolver.java`
- `email-google/.../EmailGoogleBeans.java`
- `document-google/.../DocumentGoogleBeans.java`

- [ ] **Step 7: Run all three module tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl contacts-google,email-google,document-google`
Expected: ALL PASS

- [ ] **Step 8: Commit**

```bash
git add contacts-google/ email-google/ document-google/
git commit -m "feat(#157,#158): migrate Google Contacts/Email/Document to ServiceConnectionProvider

Delete GoogleCredentialResolver (contacts), EmailGoogleBeans, DocumentGoogleBeans.
All three consumers now use ServiceConnectionProvider.getAccessToken() and
@RequiresScopes for their Google API scopes.

Refs #157, #158"
```

---

## Batch 3: GitHub + Location + full build verification

### Task 3: Migrate GitHubProjectPlatform + GoogleLocationPlatform, verify full build

**Files:**
- Modify: `project-github/pom.xml` — add platform-api dependency
- Modify: `project-github/src/main/java/io/casehub/connectors/project/github/GitHubProjectPlatform.java`
- Delete: `project-github/src/main/java/io/casehub/connectors/project/github/GitHubCredentialResolver.java` (use `ide_refactor_safe_delete`)
- Modify: `project-github/src/test/java/io/casehub/connectors/project/github/GitHubProjectPlatformTest.java`
- Modify: `location-google/pom.xml` — add platform-api dependency
- Modify: `location-google/src/main/java/io/casehub/connectors/location/google/GoogleLocationPlatform.java`
- Delete: `location-google/src/main/java/io/casehub/connectors/location/google/GoogleMapsKeyResolver.java` (use `ide_refactor_safe_delete`)
- Delete: `location-google/src/main/java/io/casehub/connectors/location/google/GoogleMapsConfig.java` (use `ide_refactor_safe_delete`)
- Modify: `location-google/src/test/java/io/casehub/connectors/location/google/GoogleLocationPlatformTest.java`

**Interfaces:**
- Consumes: `ServiceConnectionProvider.getAccessToken()` for GitHub, `StaticCredentialStore.find()` for Google Maps
- Produces: Fully migrated codebase — all resolver interfaces deleted, all consumers using platform types

- [ ] **Step 1: Add platform-api dependency to project-github and location-google pom.xml files**

Same dependency block as Task 1.

- [ ] **Step 2: Write failing tests for both consumers**

For GitHubProjectPlatform: test that `ServiceConnectionProvider` mock returns a token and `issues(actorId, tenancyId)` uses it:

```java
@Test
void issues_usesServiceConnectionProvider() {
    var provider = mock(ServiceConnectionProvider.class);
    var token = new ServiceAccessToken("ghp_test123",
        Instant.now().plusSeconds(3600), Set.of("repo"));
    when(provider.getAccessToken("user1", "github", "default"))
        .thenReturn(token);

    platform = new GitHubProjectPlatform(client, provider);
    var issues = platform.issues("user1", "default");
    assertNotNull(issues);
}
```

For GoogleLocationPlatform: test that `StaticCredentialStore` mock returns an API key:

```java
@Test
void placeSearch_usesStaticCredentialStore() {
    var store = mock(StaticCredentialStore.class);
    when(store.find("user1", "google-maps", "default"))
        .thenReturn(Optional.of(new StaticCredentialRecord(
            "user1", "default", "google-maps", "AIzaTest123",
            CredentialType.API_KEY, Instant.now(), Instant.now())));

    platform = new GoogleLocationPlatform(store);
    assertDoesNotThrow(() -> platform.placeSearch("user1", "default"));
}
```

Run each test to verify failure.

- [ ] **Step 3: Migrate GitHubProjectPlatform**

Replace `@Inject GitHubCredentialResolver resolver` → `@Inject ServiceConnectionProvider connectionProvider`.

Replace all `resolver.resolveToken(userId)` calls → `connectionProvider.getAccessToken(actorId, "github", tenancyId).accessToken()`.

Add `@RequiresScopes`:

```java
@RequiresScopes(provider = "github", scopes = {"repo", "project"})
```

Each capability accessor changes from `issues(String userId)` to resolving the token internally:

```java
    @Override
    public Issues issues(String userId) {
        var token = connectionProvider.getAccessToken(userId, "github", tenancyId()).accessToken();
        return new GitHubIssues(token);
    }
```

Note: tenancyId resolution depends on whether the SPI interface changes to include tenancyId or whether it's resolved from CDI context. If the SPI interface stays as `issues(String userId)`, inject a tenancy context bean alongside ServiceConnectionProvider.

- [ ] **Step 4: Migrate GoogleLocationPlatform**

Replace `GoogleMapsKeyResolver resolver` → `StaticCredentialStore credentialStore`.

Replace `contextFor(String userId)`:

```java
    private GeoApiContext contextFor(String actorId, String tenancyId) {
        var record = credentialStore.find(actorId, "google-maps", tenancyId)
            .orElseThrow(() -> new ServiceConnectionException(
                "google-maps", actorId, Set.of(), Set.of(), Set.of()));
        return contexts.computeIfAbsent(record.credential(), key ->
            new GeoApiContext.Builder().apiKey(key).build());
    }
```

No `@RequiresScopes` needed — Google Maps uses API keys, not OAuth scopes.

- [ ] **Step 5: Delete obsolete resolver files**

Use `ide_refactor_safe_delete`:
- `project-github/.../GitHubCredentialResolver.java`
- `location-google/.../GoogleMapsKeyResolver.java`
- `location-google/.../GoogleMapsConfig.java`

- [ ] **Step 6: Run module tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl project-github,location-google`
Expected: ALL PASS

- [ ] **Step 7: Full build verification**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn clean install`
Expected: BUILD SUCCESS — all modules compile and test, no references to deleted resolver types remain.

- [ ] **Step 8: Commit**

```bash
git add project-github/ location-google/
git commit -m "feat(#157): migrate GitHub + Location to ServiceConnectionProvider/StaticCredentialStore

Delete GitHubCredentialResolver, GoogleMapsKeyResolver, GoogleMapsConfig.
GitHubProjectPlatform uses ServiceConnectionProvider.getAccessToken().
GoogleLocationPlatform uses StaticCredentialStore.find() for API keys.
Add @RequiresScopes for github repo+project scopes.
Full build green — all credential resolver interfaces removed.

Refs #157"
```

---

## References

- [2026-10-06-connectionplatform-design.md] — design spec this plan implements
- `calendar-google/src/main/java/.../GoogleCalendarPlatform.java:29` — resolver-based pattern template
- `contacts-google/src/main/java/.../GoogleContactsPlatform.java:36` — resolver-based pattern
- `email-google/src/main/java/.../GoogleEmailPlatform.java:29` — raw credentials pattern
- `document-google/src/main/java/.../GoogleDocumentPlatform.java:32` — raw credentials pattern
- `project-github/src/main/java/.../GitHubProjectPlatform.java:20` — PAT resolver pattern
- `location-google/src/main/java/.../GoogleLocationPlatform.java:23` — API key resolver pattern
- `email-google/src/main/java/.../EmailGoogleBeans.java` — CDI producer to delete
- `document-google/src/main/java/.../DocumentGoogleBeans.java` — CDI producer to delete
- Platform `ServiceConnectionProvider` — `platform-api/.../api/authn/ServiceConnectionProvider.java`
- Platform `StaticCredentialStore` — `platform-api/.../api/authn/StaticCredentialStore.java`
- Platform `@RequiresScopes` — `platform-api/.../api/authn/RequiresScopes.java`
- Protocol: `credential-config-ownership.md` — credentials at call time
- Protocol: `shared-http-client.md` — HttpHelper.CLIENT
- GitHub #157, #158
