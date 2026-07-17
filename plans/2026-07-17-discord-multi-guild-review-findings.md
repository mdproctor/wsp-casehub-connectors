# Discord Multi-Guild + Review Findings Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #39 — chore(discord): minor code review findings from #30/#33/#34
**Issue group:** #39, #31

**Goal:** Make DiscordClient a stateless HTTP transport (guild-id as parameter, not field), enable multi-guild operation across all consuming layers, and resolve three code review findings.

**Architecture:** Remove `casehub.discord.guild-id` config property. DiscordClient methods accept guildId as a parameter. Consuming layers (DiscordChatPlatform, DiscordDiscovery, DiscordInboundConnector) each discover guilds via `GET /users/@me/guilds` or extract guild context from events. DiscordChannel gains a `guildId` field parsed from API responses.

**Tech Stack:** Java 21 (Java 26 JVM), Quarkus 3.32.2, WireMock, AssertJ, Awaitility

## Global Constraints

- Build: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn clean install`
- Module test: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl <module> -Dtest=<TestClass>`
- IntelliJ MCP mandatory for all `.java` edits — use `ide_edit_member`, `ide_insert_member`, `ide_replace_member`, `ide_create_file`
- `project_path`: `/Users/mdproctor/claude/casehub/connectors`
- Protocol: `HttpHelper.CLIENT` for all outbound HTTP (shared-http-client protocol)
- Protocol: paginating methods return partial results + WARNING on failure (paginating-client-fail-soft)

---

### Task 1: DiscordChannel — add guildId field

**Files:**
- Modify: `discord/src/main/java/io/casehub/connectors/discord/model/DiscordChannel.java`
- Modify: `discord/src/main/java/io/casehub/connectors/discord/DiscordClient.java` (parseChannel methods)
- Modify: `discord/src/test/java/io/casehub/connectors/discord/DiscordDiscoveryTest.java` (constructor args)

**Interfaces:**
- Produces: `DiscordChannel(String id, String guildId, String name, String topic, int type, String parentId, List<PermissionOverwrite> permissionOverwrites)` — new record shape used by all subsequent tasks
- Produces: `DiscordClient.parseChannel(JsonNode node, String guildId)` — overload for `listGuildChannels` where guild_id is known from the request but absent from the response JSON

- [ ] **Step 1: Write failing test — DiscordChannel has guildId**

Add to `DiscordClientTest`:

```java
@Test
void parseChannel_extractsGuildId() {
    // JSON from GET /channels/{id} includes guild_id
    wireMock.stubFor(get(urlEqualTo("/channels/ch1"))
            .willReturn(okJson("""
                    {"id":"ch1","name":"general","type":0,"guild_id":"g1"}
                    """)));

    final DiscordChannel channel = client.getChannel(TOKEN, "ch1");

    assertThat(channel).isNotNull();
    assertThat(channel.guildId()).isEqualTo("g1");
}

@Test
void parseChannel_nullGuildIdWhenAbsent() {
    wireMock.stubFor(get(urlEqualTo("/channels/dm1"))
            .willReturn(okJson("""
                    {"id":"dm1","name":"DM","type":1}
                    """)));

    final DiscordChannel channel = client.getChannel(TOKEN, "dm1");

    assertThat(channel).isNotNull();
    assertThat(channel.guildId()).isNull();
}
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl discord -Dtest="DiscordClientTest#parseChannel_extractsGuildId+parseChannel_nullGuildIdWhenAbsent"`
Expected: Compilation failure — `guildId()` does not exist on DiscordChannel

- [ ] **Step 3: Add guildId to DiscordChannel record**

Use `ide_edit_member` on `DiscordChannel.java`, member=`DiscordChannel`:

```java
public record DiscordChannel(String id, String guildId, String name, String topic, int type,
                             String parentId,
                             List<PermissionOverwrite> permissionOverwrites) {
    public DiscordChannel {
        permissionOverwrites = permissionOverwrites == null
                ? List.of() : List.copyOf(permissionOverwrites);
    }
}
```

- [ ] **Step 4: Update parseChannel in DiscordClient**

Use `ide_edit_member` on `DiscordClient.java`, member=`parseChannel`, parameterCount=1:

```java
private DiscordChannel parseChannel(final JsonNode node) {
    return parseChannel(node, node.has("guild_id") && !node.get("guild_id").isNull()
            ? node.get("guild_id").asText() : null);
}
```

Use `ide_insert_member` after `parseChannel` (1-param):

```java
private DiscordChannel parseChannel(final JsonNode node, final String guildId) {
    final List<PermissionOverwrite> overwrites = new ArrayList<>();
    if (node.has("permission_overwrites") && !node.get("permission_overwrites").isNull()) {
        for (final JsonNode ow : node.get("permission_overwrites")) {
            overwrites.add(new PermissionOverwrite(
                    ow.get("id").asText(),
                    ow.get("type").asInt(),
                    ow.get("allow").asLong(),
                    ow.get("deny").asLong()
            ));
        }
    }

    return new DiscordChannel(
            node.get("id").asText(),
            guildId,
            node.has("name") ? node.get("name").asText() : "",
            node.has("topic") && !node.get("topic").isNull() ? node.get("topic").asText() : null,
            node.get("type").asInt(),
            node.has("parent_id") && !node.get("parent_id").isNull() ? node.get("parent_id").asText() : null,
            overwrites
    );
}
```

- [ ] **Step 5: Fix DiscordDiscoveryTest constructors**

Update all `new DiscordChannel(...)` calls in `DiscordDiscoveryTest.java` to include `null` as guildId (second parameter). Use `ide_replace_text_in_file`:

Search: `new DiscordChannel("ch1", "general"` → Replace: `new DiscordChannel("ch1", null, "general"`
Search: `new DiscordChannel("ch2", "announcements"` → Replace: `new DiscordChannel("ch2", null, "announcements"`
(repeat for all constructors — `"th1"`, `"voice"`, `"cat"`, `"forum"`)

- [ ] **Step 6: Run all discord module tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl discord`
Expected: All tests pass

- [ ] **Step 7: Verify with ide_diagnostics**

Run `ide_diagnostics` on `DiscordChannel.java` and `DiscordClient.java` — no errors.

- [ ] **Step 8: Commit**

```bash
git add discord/src/main/java/io/casehub/connectors/discord/model/DiscordChannel.java \
        discord/src/main/java/io/casehub/connectors/discord/DiscordClient.java \
        discord/src/test/java/io/casehub/connectors/discord/DiscordDiscoveryTest.java \
        discord/src/test/java/io/casehub/connectors/discord/DiscordClientTest.java
git commit -m "feat(discord): add guildId to DiscordChannel record — Refs #31"
```

---

### Task 2: DiscordClient — configurable CDN hosts + null URL test

**Files:**
- Modify: `discord/src/main/java/io/casehub/connectors/discord/DiscordClient.java`
- Modify: `discord/src/test/java/io/casehub/connectors/discord/DiscordClientTest.java`

**Interfaces:**
- Produces: `@ConfigProperty(name = "casehub.discord.attachment.allowed-cdn-hosts")` — new config property
- Produces: `DiscordClient.init()` — `@PostConstruct` method

- [ ] **Step 1: Write failing tests**

Add to `DiscordClientTest`:

```java
@Test
void downloadAttachment_nullUrl_returnsNull() {
    final var att = new DiscordAttachment("a1", "file.png", "image/png", 100, null);

    final Attachment result = client.downloadAttachment(att);

    assertThat(result).isNull();
}
```

- [ ] **Step 2: Run test — verify it fails or passes (documenting existing behavior)**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl discord -Dtest="DiscordClientTest#downloadAttachment_nullUrl_returnsNull"`
Expected: May pass (NPE caught by existing handler) — if so, the test documents the path. If it fails, the exception handler isn't catching it.

- [ ] **Step 3: Add @PostConstruct and configurable CDN hosts**

Use `ide_insert_member` on `DiscordClient.java`, position=`after`, anchor=`allowedCdnHosts` (the field):

Replace the `allowedCdnHosts` field. Use `ide_edit_member`, member=`allowedCdnHosts`:

```java
@ConfigProperty(name = "casehub.discord.attachment.allowed-cdn-hosts",
                defaultValue = "cdn.discordapp.com,media.discordapp.net")
String allowedCdnHostsConfig;

Set<String> allowedCdnHosts;
```

Add `@PostConstruct` method using `ide_insert_member`, position=`after`, anchor=`DiscordClient` (constructor):

```java
@jakarta.annotation.PostConstruct
void init() {
    this.allowedCdnHosts = Set.of(allowedCdnHostsConfig.split(","));
}
```

Add import for `jakarta.annotation.PostConstruct`.

- [ ] **Step 4: Update test setup**

In `DiscordClientTest.setUp()`, after setting `client.maxAttachmentBytes`, add:

```java
client.allowedCdnHostsConfig = "cdn.discordapp.com,media.discordapp.net";
client.allowedCdnHosts = Set.of("cdn.discordapp.com", "media.discordapp.net");
```

Remove any existing direct `allowedCdnHosts` initialization if present.

- [ ] **Step 5: Run all discord module tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl discord`
Expected: All tests pass

- [ ] **Step 6: Commit**

```bash
git add discord/src/main/java/io/casehub/connectors/discord/DiscordClient.java \
        discord/src/test/java/io/casehub/connectors/discord/DiscordClientTest.java
git commit -m "feat(discord): configurable CDN hosts + null URL test — Closes #39"
```

---

### Task 3: DiscordClient — remove guildId, add guildId parameters

**Files:**
- Modify: `discord/src/main/java/io/casehub/connectors/discord/DiscordClient.java`
- Modify: `discord/src/main/java/io/casehub/connectors/discord/DiscordDiscovery.java` (call-site fix)
- Modify: `discord/src/test/java/io/casehub/connectors/discord/DiscordClientTest.java`
- Modify: `discord/src/test/java/io/casehub/connectors/discord/DiscordDiscoveryTest.java`

**Interfaces:**
- Produces: `listGuildChannels(String token, String guildId)` — returns channels with guildId set
- Produces: `createChannel(String token, String guildId, String name, String topic, int type, boolean nsfw, boolean isPrivate)`
- Produces: `listGuildMembers(String token, String guildId, int limit, String afterUserId)`
- Produces: `getGuildMember(String token, String guildId, String userId)`
- Produces: `getGuild(String token, String guildId, boolean withCounts)`
- Removes: `guildId()` accessor, `guildId` field

- [ ] **Step 1: Update DiscordClientTest — add guildId parameter to method calls**

Use `ide_replace_text_in_file` on `DiscordClientTest.java`:

Replace all calls to guild-scoped methods with the new signatures:
- `client.listGuildChannels(TOKEN)` → `client.listGuildChannels(TOKEN, GUILD_ID)`
- `client.createChannel(TOKEN, "test",` → `client.createChannel(TOKEN, GUILD_ID, "test",`
- `client.createChannel(TOKEN, "private",` → `client.createChannel(TOKEN, GUILD_ID, "private",`
- `client.listGuildMembers(TOKEN, 100, null)` → `client.listGuildMembers(TOKEN, GUILD_ID, 100, null)`
- `client.getGuildMember(TOKEN, "u1")` → `client.getGuildMember(TOKEN, GUILD_ID, "u1")`
- `client.getGuild(TOKEN, true)` → `client.getGuild(TOKEN, GUILD_ID, true)`

Remove `client.guildId = GUILD_ID;` from setUp().

Remove `guildId_returnsConfiguredValue` test.

- [ ] **Step 2: Run tests to verify they fail**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl discord -Dtest=DiscordClientTest`
Expected: Compilation failure — methods don't accept guildId parameter yet

- [ ] **Step 3: Change method signatures in DiscordClient**

For each of the 5 methods, use `ide_edit_member` to add `guildId` parameter and replace `this.guildId` with the parameter in the method body:

**listGuildChannels** — add `final String guildId` parameter. Change URI from `apiBaseUrl + "/guilds/" + guildId + "/channels"`. After building the result list, set guildId on each channel by using `parseChannel(ch, guildId)` instead of `parseChannel(ch)`.

**createChannel** — add `final String guildId` parameter (after `token`). Update URI and the permission_overwrites `overwrite.put("id", guildId)`.

**listGuildMembers** — add `final String guildId` parameter (after `token`). Update URI.

**getGuildMember** — add `final String guildId` parameter (after `token`). Update URI.

**getGuild** — add `final String guildId` parameter (after `token`). Update URI.

Remove the `guildId` field (`@ConfigProperty` annotation and field declaration).

Remove the `guildId()` accessor method.

- [ ] **Step 4: Fix listGuildChannels to use parseChannel overload**

In `listGuildChannels`, change `result.add(parseChannel(ch))` to `result.add(parseChannel(ch, guildId))` so each returned channel carries the guild ID from the request parameter.

- [ ] **Step 5: Fix DiscordDiscovery call site**

DiscordDiscovery calls `client.listGuildChannels(token)`. Update to `client.listGuildChannels(token, guildId)` — the `guildId` field still exists on DiscordDiscovery from its own `@ConfigProperty` injection.

- [ ] **Step 6: Update DiscordDiscoveryTest StubDiscordClient**

Update the `StubDiscordClient` inner class override signature:
```java
@Override
public List<DiscordChannel> listGuildChannels(final String token, final String guildId) {
    return channels;
}
```

- [ ] **Step 7: Run all discord module tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl discord`
Expected: All tests pass

- [ ] **Step 8: Verify with ide_diagnostics**

Run `ide_diagnostics` on `DiscordClient.java`, `DiscordDiscovery.java`.

- [ ] **Step 9: Commit**

```bash
git add discord/src/main/java/io/casehub/connectors/discord/DiscordClient.java \
        discord/src/main/java/io/casehub/connectors/discord/DiscordDiscovery.java \
        discord/src/test/java/io/casehub/connectors/discord/DiscordClientTest.java \
        discord/src/test/java/io/casehub/connectors/discord/DiscordDiscoveryTest.java
git commit -m "refactor(discord): remove guildId from DiscordClient — guild as parameter — Refs #31"
```

---

### Task 4: DiscordClient — add listBotGuilds method

**Files:**
- Modify: `discord/src/main/java/io/casehub/connectors/discord/DiscordClient.java`
- Modify: `discord/src/main/java/io/casehub/connectors/discord/model/DiscordGuild.java` (if id/name parsing changes needed)
- Modify: `discord/src/test/java/io/casehub/connectors/discord/DiscordClientTest.java`

**Interfaces:**
- Produces: `List<DiscordGuild> listBotGuilds(String token)` — paginated, returns `null` on first-page failure, empty list for zero guilds, partial list on mid-page error

- [ ] **Step 1: Write failing tests**

Add to `DiscordClientTest`:

```java
@Test
void listBotGuilds_success() {
    wireMock.stubFor(get(urlEqualTo("/users/@me/guilds?limit=200"))
            .willReturn(okJson("""
                    [{"id":"g1","name":"Guild One"},
                     {"id":"g2","name":"Guild Two"}]
                    """)));
    // Empty second page stops pagination
    wireMock.stubFor(get(urlMatching("/users/@me/guilds\\?limit=200&after=g2"))
            .willReturn(okJson("[]")));

    final List<DiscordGuild> guilds = client.listBotGuilds(TOKEN);

    assertThat(guilds).hasSize(2);
    assertThat(guilds.get(0).id()).isEqualTo("g1");
    assertThat(guilds.get(0).name()).isEqualTo("Guild One");
    assertThat(guilds.get(1).id()).isEqualTo("g2");
}

@Test
void listBotGuilds_nullOnFirstPageFailure() {
    wireMock.stubFor(get(urlEqualTo("/users/@me/guilds?limit=200"))
            .willReturn(aResponse().withStatus(401)));

    final List<DiscordGuild> guilds = client.listBotGuilds(TOKEN);

    assertThat(guilds).isNull();
}

@Test
void listBotGuilds_failSoftOnPage2() {
    wireMock.stubFor(get(urlEqualTo("/users/@me/guilds?limit=200"))
            .willReturn(okJson("""
                    [{"id":"g1","name":"Guild One"}]
                    """)));
    wireMock.stubFor(get(urlMatching("/users/@me/guilds\\?limit=200&after=g1"))
            .willReturn(aResponse().withStatus(500)));

    final List<DiscordGuild> guilds = client.listBotGuilds(TOKEN);

    assertThat(guilds).hasSize(1);
    assertThat(guilds.get(0).id()).isEqualTo("g1");
}

@Test
void listBotGuilds_emptyListForNoGuilds() {
    wireMock.stubFor(get(urlEqualTo("/users/@me/guilds?limit=200"))
            .willReturn(okJson("[]")));

    final List<DiscordGuild> guilds = client.listBotGuilds(TOKEN);

    assertThat(guilds).isEmpty();
}
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl discord -Dtest="DiscordClientTest#listBotGuilds_success+listBotGuilds_nullOnFirstPageFailure+listBotGuilds_failSoftOnPage2+listBotGuilds_emptyListForNoGuilds"`
Expected: Compilation failure — `listBotGuilds` does not exist

- [ ] **Step 3: Implement listBotGuilds**

Use `ide_insert_member` on `DiscordClient.java`, position=`after`, anchor=`getGatewayUrl`:

```java
/**
 * Lists all guilds the bot is a member of.
 *
 * <p>Paginates using cursor-based pagination ({@code after} parameter, 200 per page,
 * {@value MAX_PAGES} page cap). Returns {@code null} on first-page failure (matching
 * {@link #getGuild} error pattern), empty list for a valid response with zero guilds,
 * partial list on mid-page error (matching {@link #listGuildMembers} fail-soft pattern).
 *
 * @param token bot token
 * @return list of guilds, or {@code null} on first-page failure
 */
public List<DiscordGuild> listBotGuilds(final String token) {
    final List<DiscordGuild> accumulated = new ArrayList<>();
    String afterId = null;
    int pageNum = 0;

    while (pageNum < MAX_PAGES) {
        final String query = afterId == null
                ? "?limit=200"
                : "?limit=200&after=" + URLEncoder.encode(afterId, StandardCharsets.UTF_8);

        try {
            final HttpRequest request = HttpRequest.newBuilder()
                    .uri(URI.create(apiBaseUrl + "/users/@me/guilds" + query))
                    .header("Authorization", "Bot " + token)
                    .timeout(REQUEST_TIMEOUT)
                    .GET()
                    .build();

            final HttpResponse<String> response =
                    HttpHelper.CLIENT.send(request, HttpResponse.BodyHandlers.ofString());

            if (response.statusCode() != 200) {
                if (pageNum == 0) {
                    LOG.warning("DiscordClient: listBotGuilds HTTP " + response.statusCode());
                    return null;
                }
                LOG.warning(String.format(
                        "DiscordClient: listBotGuilds stopped after %d page(s) — returned %d guilds: HTTP %d",
                        pageNum, accumulated.size(), response.statusCode()));
                return List.copyOf(accumulated);
            }

            final ArrayNode guilds = (ArrayNode) mapper.readTree(response.body());
            if (guilds.isEmpty()) {
                break;
            }

            for (final JsonNode guildNode : guilds) {
                accumulated.add(new DiscordGuild(
                        guildNode.get("id").asText(),
                        guildNode.get("name").asText(),
                        null));
            }

            pageNum++;
            afterId = guilds.get(guilds.size() - 1).get("id").asText();

        } catch (final InterruptedException e) {
            Thread.currentThread().interrupt();
            LOG.warning(String.format(
                    "DiscordClient: listBotGuilds interrupted after %d page(s) — returned %d guilds",
                    pageNum, accumulated.size()));
            return pageNum == 0 ? null : List.copyOf(accumulated);
        } catch (final Exception e) {
            if (pageNum == 0) {
                LOG.warning("DiscordClient: listBotGuilds error — " + e.getMessage());
                return null;
            }
            LOG.warning(String.format(
                    "DiscordClient: listBotGuilds error after %d page(s) — returned %d guilds: %s",
                    pageNum, accumulated.size(), e.getMessage()));
            return List.copyOf(accumulated);
        }
    }

    return List.copyOf(accumulated);
}
```

- [ ] **Step 4: Run all discord module tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl discord`
Expected: All tests pass

- [ ] **Step 5: Commit**

```bash
git add discord/src/main/java/io/casehub/connectors/discord/DiscordClient.java \
        discord/src/test/java/io/casehub/connectors/discord/DiscordClientTest.java
git commit -m "feat(discord): add listBotGuilds with cursor pagination — Refs #31"
```

---

### Task 5: DiscordDiscovery — multi-guild aggregation

**Files:**
- Modify: `discord/src/main/java/io/casehub/connectors/discord/DiscordDiscovery.java`
- Modify: `discord/src/test/java/io/casehub/connectors/discord/DiscordDiscoveryTest.java`

**Interfaces:**
- Consumes: `DiscordClient.listBotGuilds(String token)` from Task 4
- Consumes: `DiscordClient.listGuildChannels(String token, String guildId)` from Task 3

- [ ] **Step 1: Write failing tests for multi-guild discovery**

Rewrite `StubDiscordClient` in `DiscordDiscoveryTest` to support `listBotGuilds` and guild-aware `listGuildChannels`:

```java
private static class StubDiscordClient extends DiscordClient {
    private final List<DiscordGuild> guilds;
    private final Map<String, List<DiscordChannel>> channelsByGuild;

    StubDiscordClient() {
        this(null, Map.of());
    }

    StubDiscordClient(final List<DiscordGuild> guilds,
                      final Map<String, List<DiscordChannel>> channelsByGuild) {
        this.guilds = guilds;
        this.channelsByGuild = channelsByGuild;
    }

    @Override
    public List<DiscordGuild> listBotGuilds(final String token) {
        return guilds;
    }

    @Override
    public List<DiscordChannel> listGuildChannels(final String token, final String guildId) {
        return channelsByGuild.getOrDefault(guildId, List.of());
    }
}
```

Add tests:

```java
@Test
void discover_multiGuild_aggregatesChannels() {
    final var guilds = List.of(
            new DiscordGuild("g1", "Guild 1", null),
            new DiscordGuild("g2", "Guild 2", null));
    final var channels = Map.of(
            "g1", List.of(new DiscordChannel("ch1", "g1", "general", null, 0, null, List.of())),
            "g2", List.of(new DiscordChannel("ch2", "g2", "lobby", null, 0, null, List.of())));
    final var discovery = new DiscordDiscovery(new StubDiscordClient(guilds, channels), TOKEN);
    final var result = discovery.discover();

    assertThat(result).hasSize(2);
    assertThat(result).extracting(DiscoveredTarget::id).containsExactlyInAnyOrder("ch1", "ch2");
}

@Test
void discover_nullFromListBotGuilds_returnsEmpty() {
    final var discovery = new DiscordDiscovery(new StubDiscordClient(null, Map.of()), TOKEN);
    final var result = discovery.discover();
    assertThat(result).isEmpty();
}

@Test
void discover_emptyGuildList_returnsEmpty() {
    final var discovery = new DiscordDiscovery(new StubDiscordClient(List.of(), Map.of()), TOKEN);
    final var result = discovery.discover();
    assertThat(result).isEmpty();
}
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl discord -Dtest=DiscordDiscoveryTest`
Expected: Compilation failure — constructor signature mismatch (still takes guildId)

- [ ] **Step 3: Rewrite DiscordDiscovery**

Use `ide_edit_member` on `DiscordDiscovery.java`:

Remove `guildId` field and constructor parameter. Change `discover()` to call `listBotGuilds` and iterate guilds:

```java
@ApplicationScoped
public class DiscordDiscovery implements ConnectorDiscovery {

    public static final String ID = "discord";
    public static final Set<Integer> TEXT_CHANNEL_TYPES = Set.of(0, 5, 10, 11, 12);

    private final DiscordClient client;
    private final String token;

    @Inject
    DiscordDiscovery(final DiscordClient client,
                     @ConfigProperty(name = "casehub.discord.token",
                                     defaultValue = "") final String token) {
        this.client = client;
        this.token = token;
    }

    @Override
    public String id() {
        return ID;
    }

    @Override
    public List<DiscoveredTarget> discover() {
        if (token.isBlank()) {
            return List.of();
        }
        final List<DiscordGuild> guilds = client.listBotGuilds(token);
        if (guilds == null || guilds.isEmpty()) {
            return List.of();
        }
        final List<DiscoveredTarget> targets = new ArrayList<>();
        for (final DiscordGuild guild : guilds) {
            final var channels = client.listGuildChannels(token, guild.id());
            if (channels != null) {
                channels.stream()
                        .filter(ch -> TEXT_CHANNEL_TYPES.contains(ch.type()))
                        .map(ch -> new DiscoveredTarget(ch.id(), "#" + ch.name()))
                        .forEach(targets::add);
            }
        }
        return targets;
    }
}
```

Add imports: `java.util.ArrayList`, `io.casehub.connectors.discord.model.DiscordGuild`.

- [ ] **Step 4: Update existing tests**

Update the single-guild tests to use the new `StubDiscordClient` constructor. The old tests that passed `GUILD_ID` as constructor arg now pass just `TOKEN`:

```java
@Test
void discover_blankTokenReturnsEmpty() {
    final var guilds = List.of(new DiscordGuild("g1", "G1", null));
    final var discovery = new DiscordDiscovery(
            new StubDiscordClient(guilds, Map.of()), "");
    assertThat(discovery.discover()).isEmpty();
}
```

Remove `discover_blankGuildIdReturnsEmpty` — guildId is no longer a constructor parameter.

Update remaining tests to use new StubDiscordClient constructor pattern.

- [ ] **Step 5: Run all discord module tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl discord`
Expected: All tests pass

- [ ] **Step 6: Commit**

```bash
git add discord/src/main/java/io/casehub/connectors/discord/DiscordDiscovery.java \
        discord/src/test/java/io/casehub/connectors/discord/DiscordDiscoveryTest.java
git commit -m "feat(discord): multi-guild discovery — iterate all bot guilds — Refs #31"
```

---

### Task 6: DiscordInboundConnector — event-driven guild context

**Files:**
- Modify: `chat-discord/src/main/java/io/casehub/connectors/chat/discord/DiscordInboundConnector.java`
- Modify: `chat-discord/src/test/java/io/casehub/connectors/chat/discord/DiscordInboundConnectorTest.java`
- Modify: `chat-discord/src/test/java/io/casehub/connectors/chat/discord/DiscordChatPlatformTest.java` (inbound tests use guild-id in constructor)

**Interfaces:**
- Consumes: `DiscordClient` — unchanged channel-scoped methods
- Produces: `metadata["discord-guild-id"]` extracted from event data instead of config

- [ ] **Step 1: Write failing test — guild_id from event data**

Add to `DiscordInboundConnectorTest`:

```java
@Test
void messageWithGuildId_extractedFromEventData() throws Exception {
    final JsonNode data = MAPPER.readTree("""
            {"id":"msg1","channel_id":"ch1","content":"hello",
             "guild_id":"guild-from-event",
             "type":0,"author":{"id":"u1","username":"user1","bot":false},
             "attachments":[]}
            """);

    connector.handleEvent("MESSAGE_CREATE", data, received::add);

    Awaitility.await().atMost(2, TimeUnit.SECONDS)
            .until(() -> !received.isEmpty());

    assertThat(received.get(0).metadata().get("discord-guild-id"))
            .isEqualTo("guild-from-event");
}

@Test
void messageWithoutGuildId_setsUnknown() throws Exception {
    final JsonNode data = MAPPER.readTree("""
            {"id":"msg1","channel_id":"ch1","content":"dm",
             "type":0,"author":{"id":"u1","username":"user1","bot":false},
             "attachments":[]}
            """);

    connector.handleEvent("MESSAGE_CREATE", data, received::add);

    Awaitility.await().atMost(2, TimeUnit.SECONDS)
            .until(() -> !received.isEmpty());

    assertThat(received.get(0).metadata().get("discord-guild-id"))
            .isEqualTo("unknown");
}
```

- [ ] **Step 2: Remove guildId from DiscordInboundConnector**

Use `ide_edit_member` on constructor and fields:

Remove `guildId` field. Remove `guildId` from constructor parameters. Remove `guildId.isBlank()` from `start()` activation guard — only check `token.isBlank()`.

In `deliverMessage()`, replace:
```java
metadata.put("discord-guild-id", guildId);
```
with:
```java
final String eventGuildId = data.has("guild_id") ? data.get("guild_id").asText() : "unknown";
metadata.put("discord-guild-id", eventGuildId);
```

Note: `deliverMessage` needs the `data` parameter. Currently it doesn't receive it — it's called from `handleMessageCreate`. Thread the `data` JsonNode through: add it as a parameter to `deliverMessage`.

- [ ] **Step 3: Update test setup**

In `DiscordInboundConnectorTest.setup()`, remove `guildId` from constructor:
```java
connector = new DiscordInboundConnector(client, new DiscordGatewayPresenceCache(), "test-token");
```

Remove `setField(client, "guildId", "guild1")` — DiscordClient no longer has guildId field.

In `DiscordChatPlatformTest`, update all `new DiscordInboundConnector(...)` calls to remove the guildId parameter.

Update existing test assertions: `inbound_messageCreateFiresEvent` test asserts `metadata.get("discord-guild-id")` equals `"guild-123"`. Add `"guild_id":"guild-123"` to the test JSON data so the event extraction finds it. Update assertion.

Remove `inbound_blankGuildIdConnectorInactive` test — guildId is no longer checked.

- [ ] **Step 4: Run chat-discord module tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl chat-discord -Dtest="DiscordInboundConnectorTest"`
Expected: All tests pass (DiscordChatPlatformTest may not compile yet — that's Task 7)

- [ ] **Step 5: Commit**

```bash
git add chat-discord/src/main/java/io/casehub/connectors/chat/discord/DiscordInboundConnector.java \
        chat-discord/src/test/java/io/casehub/connectors/chat/discord/DiscordInboundConnectorTest.java \
        chat-discord/src/test/java/io/casehub/connectors/chat/discord/DiscordChatPlatformTest.java
git commit -m "fix(discord): extract guild_id from event data, not config — Refs #31"
```

---

### Task 7: DiscordChatPlatform — multi-guild

**Files:**
- Modify: `chat-discord/src/main/java/io/casehub/connectors/chat/discord/DiscordChatPlatform.java`
- Modify: `chat-discord/src/test/java/io/casehub/connectors/chat/discord/DiscordChatPlatformTest.java`

**Interfaces:**
- Consumes: `DiscordClient.listBotGuilds(String token)` from Task 4
- Consumes: `DiscordClient.listGuildChannels(String token, String guildId)` from Task 3
- Consumes: `DiscordClient.listGuildMembers(String token, String guildId, int limit, String afterUserId)` from Task 3
- Consumes: `DiscordClient.getGuild(String token, String guildId, boolean withCounts)` from Task 3
- Consumes: `DiscordClient.createChannel(String token, String guildId, ...)` from Task 3
- Consumes: `DiscordClient.getChannel(String token, String channelId)` — unchanged
- Consumes: `DiscordChannel.guildId()` from Task 1

- [ ] **Step 1: Update DiscordChatPlatform constructor — remove guildId**

Use `ide_edit_member` on `DiscordChatPlatform.java`, member=constructor:

```java
@Inject
public DiscordChatPlatform(
        final DiscordClient client,
        final DiscordGatewayPresenceCache presenceCache,
        @ConfigProperty(name = "casehub.discord.token", defaultValue = "") final String token) {
    this.client = client;
    this.presenceCache = presenceCache;
    this.token = token;
}
```

Remove the `guildId` field.

- [ ] **Step 2: Add guild fields and rewrite init()**

Add fields using `ide_insert_member`:

```java
private List<DiscordGuild> guilds = List.of();
private Map<String, DiscordGuild> guildDetails = Map.of();
private final Map<String, String> channelToGuild = new java.util.concurrent.ConcurrentHashMap<>();
private Set<Class<?>> activeCapabilities = Set.of();
```

Rewrite `init()` using `ide_edit_member`:

```java
@PostConstruct
void init() {
    if (token.isBlank()) {
        LOG.warning("discord: token not configured, platform inactive");
        initDegraded();
        return;
    }

    final List<DiscordGuild> discovered = client.listBotGuilds(token);
    if (discovered == null) {
        LOG.severe("discord: failed to discover guilds — check token and network connectivity");
        initDegraded();
        return;
    }
    if (discovered.isEmpty()) {
        LOG.warning("discord: bot not added to any guild, platform inactive");
        initDegraded();
        return;
    }

    this.guilds = discovered;
    this.guildDetails = new java.util.HashMap<>();
    for (final DiscordGuild g : guilds) {
        final DiscordGuild details = client.getGuild(token, g.id(), true);
        guildDetails.put(g.id(), details != null ? details : g);
    }
    this.activeCapabilities = NATIVE_CAPABILITIES;

    this.messaging = this::sendMessage;
    this.threading = this::sendReply;
    this.discovery = this::listChannels;
    this.reactions = new DiscordReactions();
    this.presence = new DiscordPresence();
    this.members = this::listMembers;
    this.channelManagement = new DiscordChannelManagement();
    this.messageHistory = this::getMessageHistory;
}

private void initDegraded() {
    this.activeCapabilities = Set.of();
    this.messaging = (channel, content) -> SendResult.failure("Discord not configured");
    this.threading = new ChannelFallbackThreading(this.messaging);
    this.discovery = new EmptyDiscovery();
    this.reactions = new NoOpReactions();
    this.presence = new UnknownPresence();
    this.members = new EmptyMembers();
    this.channelManagement = new NoOpChannelManagement();
    this.messageHistory = new EmptyMessageHistory();
}
```

- [ ] **Step 3: Update supports()**

Use `ide_edit_member`:

```java
@Override
public boolean supports(final Class<?> capability) {
    return activeCapabilities.contains(capability);
}
```

- [ ] **Step 4: Rewrite listChannels for multi-guild**

Use `ide_edit_member` on `listChannels`:

```java
private List<Channel> listChannels() {
    final List<Channel> allChannels = new ArrayList<>();
    for (final DiscordGuild guild : guilds) {
        final List<DiscordChannel> channels = client.listGuildChannels(token, guild.id());
        final DiscordGuild details = guildDetails.get(guild.id());
        final Integer memberCount = details != null ? details.approximateMemberCount() : null;

        for (final DiscordChannel ch : channels) {
            if (isTextChannel(ch.type())) {
                channelToGuild.put(ch.id(), guild.id());
                allChannels.add(toChannel(ch, memberCount));
            }
        }
    }
    return allChannels;
}
```

- [ ] **Step 5: Rewrite listMembers for guild resolution**

Use `ide_edit_member` on `listMembers`:

```java
private List<Member> listMembers(final ChatChannelRef channel) {
    final String resolvedGuildId = resolveGuildId(channel.id());
    final List<DiscordMember> members = client.listGuildMembers(token, resolvedGuildId, 1000, null);
    return members.stream().map(this::toMember).toList();
}

private String resolveGuildId(final String channelId) {
    if (guilds.size() == 1) {
        return guilds.get(0).id();
    }
    final String cached = channelToGuild.get(channelId);
    if (cached != null) {
        return cached;
    }
    final DiscordChannel ch = client.getChannel(token, channelId);
    if (ch != null && ch.guildId() != null) {
        channelToGuild.put(channelId, ch.guildId());
        return ch.guildId();
    }
    return guilds.get(0).id();
}
```

- [ ] **Step 6: Rewrite DiscordChannelManagement for multi-guild**

Use `ide_edit_member` on `DiscordChannelManagement`:

```java
private class DiscordChannelManagement implements ChannelManagement {
    @Override
    public Channel create(final String name, final String topic, final String description, final boolean isPrivate) {
        if (guilds.size() != 1) {
            throw new IllegalStateException(
                    "Multiple guilds — channel creation requires disambiguation");
        }
        final String targetGuildId = guilds.get(0).id();
        final DiscordChannel dc = client.createChannel(token, targetGuildId, name, topic, 0, false, isPrivate);
        if (dc == null) {
            throw new IllegalStateException("Channel creation failed");
        }
        return toChannel(dc, null);
    }

    @Override
    public void delete(final String channelId) {
        client.deleteChannel(token, channelId);
    }

    @Override
    public Optional<Channel> find(final String channelId) {
        final DiscordChannel dc = client.getChannel(token, channelId);
        if (dc == null) {
            return Optional.empty();
        }
        if (dc.guildId() != null) {
            channelToGuild.put(dc.id(), dc.guildId());
        }
        return Optional.of(toChannel(dc, null));
    }
}
```

- [ ] **Step 7: Update isPrivateChannel to use channel.guildId()**

Use `ide_edit_member` on `isPrivateChannel`:

```java
private boolean isPrivateChannel(final DiscordChannel channel) {
    final String everyoneRoleId = channel.guildId();
    if (everyoneRoleId == null) {
        return false;
    }
    return channel.permissionOverwrites().stream()
            .anyMatch(po -> po.type() == 0
                    && po.id().equals(everyoneRoleId)
                    && (po.deny() & VIEW_CHANNEL_BIT) != 0);
}
```

- [ ] **Step 8: Update test setUp — remove guildId, add WireMock stubs for guild discovery**

Rewrite `setUp()`:

```java
@BeforeEach
void setUp() throws Exception {
    wireMock.resetAll();
    client = new DiscordClient();

    var apiBaseUrlField = DiscordClient.class.getDeclaredField("apiBaseUrl");
    apiBaseUrlField.setAccessible(true);
    apiBaseUrlField.set(client, "http://localhost:" + wireMock.port());

    var allowedCdnHostsConfig = DiscordClient.class.getDeclaredField("allowedCdnHostsConfig");
    allowedCdnHostsConfig.setAccessible(true);
    allowedCdnHostsConfig.set(client, "cdn.discordapp.com,media.discordapp.net,localhost");

    var maxAttachmentBytesField = DiscordClient.class.getDeclaredField("maxAttachmentBytes");
    maxAttachmentBytesField.setAccessible(true);
    maxAttachmentBytesField.set(client, 8388608L);

    // Trigger @PostConstruct manually
    client.getClass().getDeclaredMethod("init").invoke(client);

    // Stub guild discovery — single guild for most tests
    wireMock.stubFor(get(urlEqualTo("/users/@me/guilds?limit=200"))
            .willReturn(okJson("""
                    [{"id":"test-guild-123","name":"Test Guild"}]
                    """)));
    wireMock.stubFor(get(urlMatching("/users/@me/guilds\\?limit=200&after=.*"))
            .willReturn(okJson("[]")));
    wireMock.stubFor(get(urlEqualTo("/guilds/test-guild-123?with_counts=true"))
            .willReturn(okJson("""
                    {"id":"test-guild-123","name":"Test Guild","approximate_member_count":42}
                    """)));

    presenceCache = new DiscordGatewayPresenceCache();
    platform = new DiscordChatPlatform(client, presenceCache, "test-token");
    platform.init();
}
```

- [ ] **Step 9: Add multi-guild specific tests**

```java
@Test
void discovery_multiGuild_aggregatesChannels() throws Exception {
    wireMock.resetAll();
    // Re-stub with two guilds
    wireMock.stubFor(get(urlEqualTo("/users/@me/guilds?limit=200"))
            .willReturn(okJson("""
                    [{"id":"g1","name":"Guild 1"},{"id":"g2","name":"Guild 2"}]
                    """)));
    wireMock.stubFor(get(urlMatching("/users/@me/guilds\\?limit=200&after=.*"))
            .willReturn(okJson("[]")));
    wireMock.stubFor(get(urlEqualTo("/guilds/g1?with_counts=true"))
            .willReturn(okJson("""
                    {"id":"g1","name":"Guild 1","approximate_member_count":10}
                    """)));
    wireMock.stubFor(get(urlEqualTo("/guilds/g2?with_counts=true"))
            .willReturn(okJson("""
                    {"id":"g2","name":"Guild 2","approximate_member_count":20}
                    """)));
    wireMock.stubFor(get(urlEqualTo("/guilds/g1/channels"))
            .willReturn(okJson("""
                    [{"id":"ch-g1","name":"general","type":0}]
                    """)));
    wireMock.stubFor(get(urlEqualTo("/guilds/g2/channels"))
            .willReturn(okJson("""
                    [{"id":"ch-g2","name":"lobby","type":0}]
                    """)));

    final var multiPlatform = new DiscordChatPlatform(client, presenceCache, "test-token");
    multiPlatform.init();

    List<Channel> channels = multiPlatform.discovery().listChannels();
    assertThat(channels).hasSize(2);
    assertThat(channels).extracting(Channel::name).containsExactlyInAnyOrder("general", "lobby");
    assertThat(channels.stream().filter(c -> c.name().equals("general")).findFirst().get().memberCount()).isEqualTo(10);
    assertThat(channels.stream().filter(c -> c.name().equals("lobby")).findFirst().get().memberCount()).isEqualTo(20);
}

@Test
void channelManagement_createThrowsOnMultiGuild() throws Exception {
    wireMock.resetAll();
    wireMock.stubFor(get(urlEqualTo("/users/@me/guilds?limit=200"))
            .willReturn(okJson("""
                    [{"id":"g1","name":"G1"},{"id":"g2","name":"G2"}]
                    """)));
    wireMock.stubFor(get(urlMatching("/users/@me/guilds\\?limit=200&after=.*"))
            .willReturn(okJson("[]")));
    wireMock.stubFor(get(urlEqualTo("/guilds/g1?with_counts=true"))
            .willReturn(okJson("""{"id":"g1","name":"G1","approximate_member_count":5}""")));
    wireMock.stubFor(get(urlEqualTo("/guilds/g2?with_counts=true"))
            .willReturn(okJson("""{"id":"g2","name":"G2","approximate_member_count":3}""")));

    final var multiPlatform = new DiscordChatPlatform(client, presenceCache, "test-token");
    multiPlatform.init();

    assertThatThrownBy(() ->
            multiPlatform.channelManagement().create("test", "topic", null, false))
            .isInstanceOf(IllegalStateException.class)
            .hasMessageContaining("Multiple guilds");
}

@Test
void init_nullFromListBotGuilds_degradedMode() throws Exception {
    wireMock.resetAll();
    wireMock.stubFor(get(urlEqualTo("/users/@me/guilds?limit=200"))
            .willReturn(aResponse().withStatus(401)));

    final var failPlatform = new DiscordChatPlatform(client, presenceCache, "test-token");
    failPlatform.init();

    assertThat(failPlatform.supports(Messaging.class)).isFalse();
    assertThat(failPlatform.discovery().listChannels()).isEmpty();
}

@Test
void init_emptyGuildList_degradedMode() throws Exception {
    wireMock.resetAll();
    wireMock.stubFor(get(urlEqualTo("/users/@me/guilds?limit=200"))
            .willReturn(okJson("[]")));

    final var emptyPlatform = new DiscordChatPlatform(client, presenceCache, "test-token");
    emptyPlatform.init();

    assertThat(emptyPlatform.supports(Messaging.class)).isFalse();
}
```

- [ ] **Step 10: Update existing tests for new constructor and API calls**

Update all `new DiscordChatPlatform(client, presenceCache, "test-token", "test-guild-123")` → `new DiscordChatPlatform(client, presenceCache, "test-token")`.

Update WireMock stubs in `discovery_listChannels` and `discovery_excludesForumChannels` — remove explicit `getGuild` stubs (now handled in setUp via init).

Update `messaging_blankTokenReturnsFailure` to use new 3-arg constructor.

Remove `discovery_blankGuildIdReturnsEmpty` test — guildId no longer in constructor.

Update `channelManagement_findDerivesIsPrivate` — add `"guild_id":"test-guild-123"` to the WireMock response JSON so `isPrivateChannel` can use `channel.guildId()`.

Update `channelManagement_createPrivateChannel` — verify the permission_overwrites use the guild's ID.

Update `members_list` — add `listGuildMembers` WireMock stubs with guild ID in URL.

- [ ] **Step 11: Run full build**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn clean install`
Expected: All modules compile and all tests pass

- [ ] **Step 12: Verify with ide_diagnostics**

Run `ide_diagnostics` on `DiscordChatPlatform.java` — no errors.

- [ ] **Step 13: Commit**

```bash
git add chat-discord/src/main/java/io/casehub/connectors/chat/discord/DiscordChatPlatform.java \
        chat-discord/src/test/java/io/casehub/connectors/chat/discord/DiscordChatPlatformTest.java
git commit -m "feat(discord): multi-guild ChatPlatform — discover guilds via API — Closes #31"
```
