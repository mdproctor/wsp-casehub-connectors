# Chat Demo UI Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add a polished casehub-pages frontend to the chat-demo module — three-column chat workspace with real-time WebSocket data, dockable side panels, dark mode, and showcase-quality UX.

**Architecture:** Custom Web Component panels hosted via casehub-pages workbench primitives (`split`, `dockBar`, `hostPanel`). Single multiplexed WebSocket at `/ws/chat` pushes all datasets. Inter-panel communication via `pages-event` DOM events. Backend broadcaster updated to emit pages-compatible wire protocol.

**Tech Stack:** Java 21 / Quarkus 3.32.2 (backend), TypeScript / esbuild / casehub-pages 0.2.0 (frontend), Quinoa (Quarkus-SPA bridge)

## Global Constraints

- Java 21 source, Java 26 JVM: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn clean install`
- Build with `-Pdemo` for chat-demo, `-Pdemo -Pui` for UI
- `mvn clean install` and `mvn clean install -Pdemo` must pass without Node.js
- casehub-pages packages referenced via `file:` paths to local clone at `../../../../casehub/pages/packages/`
- All wire protocol values are strings or null — no JSON booleans/numbers
- Column definitions require `id`, `name`, and `type` fields
- Quinoa node version pinned to `22.16.0` (GE-20260630-bf0055)
- IntelliJ MCP does not index TypeScript (GE-20260701-f1c67f) — use bash for TS files
- `@QuarkusTest` disables Quinoa (GE-20260701-c000c7) — UI needs manual testing
- Every commit references issue #28

## File Map

### Modified Files

| File | Changes |
|------|---------|
| `chat-demo/pom.xml` | Add `-Pui` profile with Quinoa dependency |
| `chat-demo/src/main/resources/application.properties` | Add Quinoa config properties |
| `chat-demo/src/main/java/.../ChatWebSocketBroadcaster.java` | Wire protocol: `type`→`op`, add `columns`/`seq`, string coercion, `membershipId`, presence snapshot |
| `chat-demo/src/main/java/.../ChatWebSocket.java` | Add `@OnTextMessage` handler for subscribe/unsubscribe |

### New Files

| File | Purpose |
|------|---------|
| `chat-demo/src/main/webui/package.json` | npm deps — pages-runtime, pages-ui via file: |
| `chat-demo/src/main/webui/esbuild.config.mjs` | esbuild bundler config |
| `chat-demo/src/main/webui/tsconfig.json` | TypeScript config |
| `chat-demo/src/main/webui/.gitignore` | Exclude node_modules/, dist/ |
| `chat-demo/src/main/webui/src/index.html` | Shell HTML |
| `chat-demo/src/main/webui/src/index.ts` | loadSite entry point — layout, datasets, theme |
| `chat-demo/src/main/webui/src/panels/channel-sidebar.ts` | Channel list Web Component |
| `chat-demo/src/main/webui/src/panels/message-list.ts` | Message display Web Component |
| `chat-demo/src/main/webui/src/panels/message-input.ts` | Compose input Web Component |
| `chat-demo/src/main/webui/src/panels/member-list.ts` | Member list + presence Web Component |
| `chat-demo/src/test/java/.../ChatWebSocketTest.java` | WebSocket wire protocol integration test |

---

### Task 1: Wire Protocol — Update ChatWebSocketBroadcaster

**Files:**
- Modify: `chat-demo/src/main/java/io/casehub/connectors/chat/demo/ChatWebSocketBroadcaster.java`
- Modify: `chat-demo/src/main/java/io/casehub/connectors/chat/demo/ChatWebSocket.java`
- Create: `chat-demo/src/test/java/io/casehub/connectors/chat/demo/ChatWebSocketTest.java`

**Interfaces:**
- Consumes: `ChatPlatform` SPI (channels, messages, members, presence), `ObjectMapper`, `WebSocketConnection`
- Produces: Wire protocol messages with `op`, `dataset`, `seq`, `columns`, `rows` fields. Four snapshot datasets on connect. `@OnTextMessage` handler on `ChatWebSocket`.

- [ ] **Step 1: Write the WebSocket integration test**

Create `ChatWebSocketTest.java`:

```java
package io.casehub.connectors.chat.demo;

import static org.assertj.core.api.Assertions.assertThat;

import java.net.URI;
import java.util.List;
import java.util.Map;
import java.util.concurrent.CompletableFuture;
import java.util.concurrent.TimeUnit;

import com.fasterxml.jackson.core.type.TypeReference;
import com.fasterxml.jackson.databind.ObjectMapper;
import io.quarkus.test.common.http.TestHTTPResource;
import io.quarkus.test.junit.QuarkusTest;
import jakarta.inject.Inject;
import jakarta.websocket.ClientEndpointConfig;
import jakarta.websocket.ContainerProvider;
import jakarta.websocket.Endpoint;
import jakarta.websocket.EndpointConfig;
import jakarta.websocket.MessageHandler;
import jakarta.websocket.Session;
import org.junit.jupiter.api.Test;

@QuarkusTest
class ChatWebSocketTest {

    @TestHTTPResource("/ws/chat")
    URI wsUri;

    @Inject
    ObjectMapper objectMapper;

    @Test
    void snapshotContainsFourDatasetsWithCorrectStructure() throws Exception {
        final var future = new CompletableFuture<String>();
        try (Session session = connectAndCapture(future)) {
            final String raw = future.get(5, TimeUnit.SECONDS);
            final List<Map<String, Object>> snapshots = objectMapper.readValue(raw,
                    new TypeReference<>() {});

            assertThat(snapshots).hasSizeGreaterThanOrEqualTo(4);

            final var datasetNames = snapshots.stream()
                    .map(s -> (String) s.get("dataset"))
                    .toList();
            assertThat(datasetNames).contains("channels", "messages", "members", "presence");

            for (final Map<String, Object> snapshot : snapshots) {
                assertThat(snapshot).containsKey("op");
                assertThat(snapshot.get("op")).isEqualTo("snapshot");
                assertThat(snapshot).containsKey("seq");
                assertThat(snapshot).containsKey("columns");
                assertThat(snapshot).containsKey("rows");

                @SuppressWarnings("unchecked")
                final var columns = (List<Map<String, Object>>) snapshot.get("columns");
                for (final Map<String, Object> col : columns) {
                    assertThat(col).containsKeys("id", "name", "type");
                }
            }
        }
    }

    @Test
    void allRowValuesAreStringsOrNull() throws Exception {
        final var future = new CompletableFuture<String>();
        try (Session session = connectAndCapture(future)) {
            final String raw = future.get(5, TimeUnit.SECONDS);
            final List<Map<String, Object>> snapshots = objectMapper.readValue(raw,
                    new TypeReference<>() {});

            for (final Map<String, Object> snapshot : snapshots) {
                @SuppressWarnings("unchecked")
                final var rows = (List<List<Object>>) snapshot.get("rows");
                for (final List<Object> row : rows) {
                    for (final Object cell : row) {
                        assertThat(cell).satisfiesAnyOf(
                                v -> assertThat(v).isNull(),
                                v -> assertThat(v).isInstanceOf(String.class));
                    }
                }
            }
        }
    }

    @Test
    void seqValuesAreMonotonicallyIncreasing() throws Exception {
        final var future = new CompletableFuture<String>();
        try (Session session = connectAndCapture(future)) {
            final String raw = future.get(5, TimeUnit.SECONDS);
            final List<Map<String, Object>> snapshots = objectMapper.readValue(raw,
                    new TypeReference<>() {});

            long lastSeq = 0;
            for (final Map<String, Object> snapshot : snapshots) {
                final long seq = Long.parseLong((String) snapshot.get("seq"));
                assertThat(seq).isGreaterThan(lastSeq);
                lastSeq = seq;
            }
        }
    }

    @Test
    void membersDatasetHasMembershipIdColumn() throws Exception {
        final var future = new CompletableFuture<String>();
        try (Session session = connectAndCapture(future)) {
            final String raw = future.get(5, TimeUnit.SECONDS);
            final List<Map<String, Object>> snapshots = objectMapper.readValue(raw,
                    new TypeReference<>() {});

            final var members = snapshots.stream()
                    .filter(s -> "members".equals(s.get("dataset")))
                    .findFirst().orElseThrow();

            @SuppressWarnings("unchecked")
            final var columns = (List<Map<String, Object>>) members.get("columns");
            assertThat(columns.get(0).get("id")).isEqualTo("membershipId");
        }
    }

    @Test
    void presenceSnapshotDeduplicatesMembersAcrossChannels() throws Exception {
        final var future = new CompletableFuture<String>();
        try (Session session = connectAndCapture(future)) {
            final String raw = future.get(5, TimeUnit.SECONDS);
            final List<Map<String, Object>> snapshots = objectMapper.readValue(raw,
                    new TypeReference<>() {});

            final var presence = snapshots.stream()
                    .filter(s -> "presence".equals(s.get("dataset")))
                    .findFirst().orElseThrow();

            @SuppressWarnings("unchecked")
            final var rows = (List<List<String>>) presence.get("rows");
            final var memberIds = rows.stream().map(r -> r.get(0)).toList();
            assertThat(memberIds).doesNotHaveDuplicates();
        }
    }

    private Session connectAndCapture(final CompletableFuture<String> future) throws Exception {
        final var container = ContainerProvider.getWebSocketContainer();
        return container.connectToServer(new Endpoint() {
            @Override
            public void onOpen(final Session session, final EndpointConfig config) {
                session.addMessageHandler(new MessageHandler.Whole<String>() {
                    @Override
                    public void onMessage(final String message) {
                        future.complete(message);
                    }
                });
            }
        }, ClientEndpointConfig.Builder.create().build(), wsUri);
    }
}
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl chat-demo -Pdemo -Dtest=ChatWebSocketTest`
Expected: FAIL — snapshot uses `type` not `op`, missing `columns`, etc.

- [ ] **Step 3: Update ChatWebSocketBroadcaster**

Apply these changes to `ChatWebSocketBroadcaster.java`:

1. Add `AtomicLong seq` field initialised to 0
2. Define column constant arrays for each dataset:
   - `CHANNEL_COLUMNS`: `id/ID/LABEL`, `name/Name/LABEL`, `topic/Topic/LABEL`, `description/Description/LABEL`, `isPrivate/Private/LABEL`
   - `MESSAGE_COLUMNS`: `channelId/Channel/LABEL`, `messageId/Message ID/LABEL`, `parentId/Parent/LABEL`, `senderId/Sender/LABEL`, `text/Text/LABEL`, `timestamp/Timestamp/DATE`
   - `MEMBER_COLUMNS`: `membershipId/Membership/LABEL`, `channelId/Channel/LABEL`, `memberId/Member/LABEL`, `displayName/Display Name/LABEL`
   - `PRESENCE_COLUMNS`: `memberId/Member/LABEL`, `status/Status/LABEL`
   - `REACTION_COLUMNS`: `messageId/Message ID/LABEL`, `emoji/Emoji/LABEL`
3. Each column is `Map.of("id", id, "name", name, "type", type)`
4. Rename `type` → `op` in all message maps
5. Add `"seq"` field (string of `seq.incrementAndGet()`) to every message
6. Add `"columns"` to all `snapshot`, `append`, and `replace` messages
7. Coerce all row values to strings: `String.valueOf(ch.isPrivate())` for booleans, `.toString()` for instants
8. Update `buildSnapshot()`:
   - Add `membershipId` as first value in member rows: `channel.ref().id() + ":" + m.ref().id()`
   - Add presence snapshot: collect unique `MemberRef` across all channels into a `LinkedHashSet`, query presence for each, build `[memberId, status.name()]` rows
   - Return array of four snapshots: channels, messages, members, presence
9. Update `broadcastMemberAppend` to include `membershipId` as first value
10. Update `broadcastMemberRemove` to use `membershipId` as key

- [ ] **Step 4: Add `@OnTextMessage` handler to ChatWebSocket**

Add to `ChatWebSocket.java`:

```java
@OnTextMessage
public void onMessage(final String message) {
    Log.debugf("WebSocket client message (ignored): %s", message);
}
```

Add `import io.quarkus.logging.Log;` and `import io.quarkus.websockets.next.OnTextMessage;`.

- [ ] **Step 5: Run tests to verify they pass**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl chat-demo -Pdemo -Dtest=ChatWebSocketTest`
Expected: All 5 tests PASS

- [ ] **Step 6: Run full test suite**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl chat-demo -Pdemo`
Expected: All tests pass (ChatResourceTest + ChatWebSocketTest + SqliteChatBackendTest)

- [ ] **Step 7: Commit**

```
git add chat-demo/src/main/java/io/casehub/connectors/chat/demo/ChatWebSocketBroadcaster.java chat-demo/src/main/java/io/casehub/connectors/chat/demo/ChatWebSocket.java chat-demo/src/test/java/io/casehub/connectors/chat/demo/ChatWebSocketTest.java
git commit -m "feat(chat-demo): align WebSocket broadcaster with pages wire protocol — Refs #28"
```

---

### Task 2: Maven + Quinoa Build Integration

**Files:**
- Modify: `chat-demo/pom.xml`
- Modify: `chat-demo/src/main/resources/application.properties`
- Create: `chat-demo/src/main/webui/package.json`
- Create: `chat-demo/src/main/webui/esbuild.config.mjs`
- Create: `chat-demo/src/main/webui/tsconfig.json`
- Create: `chat-demo/src/main/webui/.gitignore`
- Create: `chat-demo/src/main/webui/src/index.html`
- Create: `chat-demo/src/main/webui/src/index.ts` (minimal placeholder)

**Interfaces:**
- Consumes: casehub-pages packages via `file:` references
- Produces: Working Quinoa build pipeline; `mvn clean install -Pdemo` passes without Node.js; `mvn clean install -Pdemo -Pui` builds the frontend

- [ ] **Step 1: Add `-Pui` profile to pom.xml**

Add this profile block inside `<profiles>` (there are no existing profiles in chat-demo's pom — add the `<profiles>` element):

```xml
<profiles>
  <profile>
    <id>ui</id>
    <dependencies>
      <dependency>
        <groupId>io.quarkiverse.quinoa</groupId>
        <artifactId>quarkus-quinoa</artifactId>
        <version>2.5.3</version>
      </dependency>
    </dependencies>
  </profile>
</profiles>
```

Note: Verify `2.5.3` against Maven Central — this is the version from the getting-started guide. If a newer version exists, use it.

- [ ] **Step 2: Add Quinoa properties to application.properties**

Append to `chat-demo/src/main/resources/application.properties`:

```properties
quarkus.quinoa.build-dir=dist
quarkus.quinoa.package-manager-install=true
quarkus.quinoa.package-manager-install.node-version=22.16.0
quarkus.quinoa.enable-spa-routing=true
```

- [ ] **Step 3: Create webui scaffolding**

Create `chat-demo/src/main/webui/package.json`:

```json
{
  "name": "chat-demo-ui",
  "private": true,
  "scripts": {
    "build": "node esbuild.config.mjs",
    "dev": "node esbuild.config.mjs --watch",
    "typecheck": "tsc --noEmit"
  },
  "dependencies": {
    "@casehubio/pages-runtime": "file:../../../../../../../casehub/pages/packages/pages-runtime",
    "@casehubio/pages-ui": "file:../../../../../../../casehub/pages/packages/pages-ui"
  },
  "devDependencies": {
    "esbuild": "^0.25.0",
    "typescript": "^5.6.0"
  }
}
```

Note: The `file:` path resolves from `chat-demo/src/main/webui/` → `../../../../../../..` reaches the user's clone root, then `casehub/pages/packages/`. Verify this path is correct by counting directory levels: `webui/` → `main/` → `src/` → `chat-demo/` → `connectors/` → `casehub/` → `claude/` = 7 levels up, then `casehub/pages/packages/pages-runtime`.

Create `chat-demo/src/main/webui/esbuild.config.mjs` (copy from template):

```javascript
import { build, context } from "esbuild";

const isWatch = process.argv.includes("--watch");

const options = {
  entryPoints: ["src/index.ts"],
  bundle: true,
  outfile: "dist/app.js",
  format: "esm",
  target: "es2020",
  minify: !isWatch,
  sourcemap: isWatch,
};

if (isWatch) {
  const ctx = await context(options);
  await ctx.watch();
  console.log("Watching for changes...");
} else {
  await build(options);
}
```

Create `chat-demo/src/main/webui/tsconfig.json`:

```json
{
  "compilerOptions": {
    "target": "ES2020",
    "module": "ESNext",
    "moduleResolution": "bundler",
    "strict": true,
    "esModuleInterop": true,
    "skipLibCheck": true,
    "outDir": "dist",
    "declaration": false,
    "noEmit": true
  },
  "include": ["src"]
}
```

Create `chat-demo/src/main/webui/.gitignore`:

```
node_modules/
dist/
```

- [ ] **Step 4: Create shell HTML and minimal entry point**

Create `chat-demo/src/main/webui/src/index.html`:

```html
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <title>Chat Demo</title>
  <style>
    *, *::before, *::after { box-sizing: border-box; margin: 0; padding: 0; }
    html, body { height: 100%; overflow: hidden; }
    #app { height: 100vh; width: 100vw; }
  </style>
</head>
<body>
  <div id="app"></div>
  <script type="module" src="./app.js"></script>
</body>
</html>
```

Create `chat-demo/src/main/webui/src/index.ts` (minimal placeholder — will be filled in Task 3):

```typescript
const app = document.getElementById("app");
if (app) {
  app.textContent = "Chat Demo — loading...";
}
```

- [ ] **Step 5: Verify build without -Pui passes**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn clean install -Pdemo`
Expected: BUILD SUCCESS — Quinoa not on classpath, webui/ directory ignored

- [ ] **Step 6: Verify build with -Pui passes**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn clean install -Pdemo -Pui`
Expected: BUILD SUCCESS — Quinoa downloads Node, runs npm install + build, bundles dist/app.js

- [ ] **Step 7: Commit**

```
git add chat-demo/pom.xml chat-demo/src/main/resources/application.properties chat-demo/src/main/webui/
git commit -m "chore(chat-demo): add Quinoa build pipeline with -Pui profile — Refs #28"
```

---

### Task 3: Frontend Entry Point — Layout, Datasets, Theme

**Files:**
- Modify: `chat-demo/src/main/webui/src/index.ts`

**Interfaces:**
- Consumes: `loadSite`, `registerPanel`, `dataset` from casehub-pages; `columns`, `split`, `dockBar`, `hostPanel`, `withId` from pages-ui
- Produces: Full component tree with four WebSocket-backed datasets; dark theme; panel registrations for `channels`, `messages`, `input`, `members`

- [ ] **Step 1: Write the full entry point**

Replace `chat-demo/src/main/webui/src/index.ts` with:

```typescript
import { loadSite, registerPanel } from "@casehubio/pages-runtime";
import { columns, split, dockBar, hostPanel, withId, dataset } from "@casehubio/pages-ui";

import "./panels/channel-sidebar.js";
import "./panels/message-list.js";
import "./panels/message-input.js";
import "./panels/member-list.js";

registerPanel("channels", "chat-channel-sidebar");
registerPanel("messages", "chat-message-list");
registerPanel("input", "chat-message-input");
registerPanel("members", "chat-member-list");

const WS_URL = `ws://${window.location.host}/ws/chat`;

const chatApp = columns([0, 1],
  [dockBar("vertical", [
    { icon: "\u{1F4AC}", label: "Channels", panelId: "channel-panel", defaultOpen: true },
    { icon: "\u{1F465}", label: "Members", panelId: "member-panel", defaultOpen: true },
  ])],
  [split("horizontal", [
    withId("channel-panel", hostPanel("channels")),
    split("vertical", [
      hostPanel("messages"),
      hostPanel("input"),
    ], { ratio: [90, 10] }),
    withId("member-panel", hostPanel("members")),
  ], { ratio: [20, 60, 20] })],
);

const container = document.getElementById("app");
if (container) {
  loadSite(container, chatApp, {
    providerConfig: {
      webSocket: {},
    },
  }).then((site) => {
    site.setTheme("dark");
  }).catch(console.error);
}
```

- [ ] **Step 2: Create stub panel files so imports resolve**

Create four stub files so esbuild can resolve the imports. Each will be replaced in Tasks 4–7.

`chat-demo/src/main/webui/src/panels/channel-sidebar.ts`:
```typescript
class ChatChannelSidebar extends HTMLElement {
  connectedCallback() { this.textContent = "Channels loading..."; }
}
customElements.define("chat-channel-sidebar", ChatChannelSidebar);
```

`chat-demo/src/main/webui/src/panels/message-list.ts`:
```typescript
class ChatMessageList extends HTMLElement {
  connectedCallback() { this.textContent = "Messages loading..."; }
}
customElements.define("chat-message-list", ChatMessageList);
```

`chat-demo/src/main/webui/src/panels/message-input.ts`:
```typescript
class ChatMessageInput extends HTMLElement {
  connectedCallback() { this.textContent = "Input loading..."; }
}
customElements.define("chat-message-input", ChatMessageInput);
```

`chat-demo/src/main/webui/src/panels/member-list.ts`:
```typescript
class ChatMemberList extends HTMLElement {
  connectedCallback() { this.textContent = "Members loading..."; }
}
customElements.define("chat-member-list", ChatMemberList);
```

- [ ] **Step 3: Verify build**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn clean install -Pdemo -Pui`
Expected: BUILD SUCCESS

- [ ] **Step 4: Manual smoke test**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn quarkus:dev -pl chat-demo -Pdemo -Pui`
Open: `http://localhost:8090`
Expected: Dark-themed page with three-column layout, dock bar on left, stub text in each panel. Side panels toggle when dock icons are clicked.

- [ ] **Step 5: Commit**

```
git add chat-demo/src/main/webui/src/
git commit -m "feat(chat-demo): layout skeleton with dock bar and stub panels — Refs #28"
```

---

### Task 4: Channel Sidebar Panel

**Files:**
- Modify: `chat-demo/src/main/webui/src/panels/channel-sidebar.ts`

**Interfaces:**
- Consumes: WebSocket `channels` dataset (via custom `pages-event` with topic `ws-snapshot` and `ws-append`), `pages-event` DOM events
- Produces: `pages-event` with topic `channel-selected`, payload `{ channelId: string }`

- [ ] **Step 1: Implement the channel sidebar**

Replace `chat-demo/src/main/webui/src/panels/channel-sidebar.ts` with the full implementation. The panel:

1. Connects to the WebSocket at `/ws/chat` directly (not via pages data pipeline — panels receive raw data and manage their own state for maximum control)
2. On `snapshot` with `dataset: "channels"`, stores the channel list and renders
3. On `append` with `dataset: "channels"`, adds the new channel and re-renders
4. Auto-selects the first channel on initial snapshot via `queueMicrotask()`
5. Dispatches `pages-event` with topic `channel-selected` on click
6. Highlights the selected channel

The WebSocket connection is shared across all panels — the entry point (`index.ts`) creates a single WebSocket and dispatches messages as DOM events. Each panel listens for the events it cares about.

**Architecture decision:** Rather than each panel connecting independently, update `index.ts` to open the WebSocket and dispatch `pages-event` messages with topic `ws-data` to all panels. This avoids four separate WebSocket connections.

Update `index.ts` to add WebSocket relay after `loadSite()`:

```typescript
// After loadSite resolves, open WebSocket and relay to panels
const ws = new WebSocket(WS_URL);
ws.addEventListener("message", (event) => {
  try {
    const data = JSON.parse(event.data);
    const messages = Array.isArray(data) ? data : [data];
    for (const msg of messages) {
      document.dispatchEvent(new CustomEvent("pages-event", {
        bubbles: true,
        composed: true,
        detail: { topic: "ws-data", payload: msg },
      }));
    }
  } catch (e) {
    console.warn("Failed to parse WebSocket message:", e);
  }
});
```

Now the channel sidebar implementation:

```typescript
const STYLES = `
  :host {
    display: flex;
    flex-direction: column;
    height: 100%;
    overflow: hidden;
    font-family: var(--pages-font, system-ui, -apple-system, sans-serif);
    background: var(--pages-bg, #1a1a2e);
    color: var(--pages-text, #e0e0e0);
  }
  .header {
    padding: 12px 16px 8px;
    font-size: 11px;
    font-weight: 600;
    text-transform: uppercase;
    letter-spacing: 0.05em;
    color: var(--pages-text-muted, #999);
  }
  .channel-list {
    flex: 1;
    overflow-y: auto;
    scrollbar-width: thin;
    scrollbar-color: var(--pages-border, #3a3a5e) transparent;
  }
  .channel {
    padding: 6px 16px;
    font-size: 14px;
    cursor: pointer;
    white-space: nowrap;
    overflow: hidden;
    text-overflow: ellipsis;
    border-radius: 4px;
    margin: 1px 8px;
    transition: background 0.15s;
  }
  .channel:hover {
    background: var(--pages-bg-hover, #1e3a5f);
  }
  .channel.selected {
    background: var(--pages-bg-hover, #1e3a5f);
    color: var(--pages-accent, #7c8cf8);
    font-weight: 600;
  }
  .channel-hash {
    color: var(--pages-text-muted, #999);
    margin-right: 4px;
    font-weight: 400;
  }
`;

interface ChannelData {
  id: string;
  name: string;
  topic: string;
}

class ChatChannelSidebar extends HTMLElement {
  private channels: ChannelData[] = [];
  private selectedId = "";
  private shadow: ShadowRoot;

  constructor() {
    super();
    this.shadow = this.attachShadow({ mode: "open" });
  }

  connectedCallback(): void {
    document.addEventListener("pages-event", this.onEvent);
    this.render();
  }

  disconnectedCallback(): void {
    document.removeEventListener("pages-event", this.onEvent);
  }

  private onEvent = (e: Event): void => {
    const { topic, payload } = (e as CustomEvent).detail;
    if (topic !== "ws-data") return;
    const msg = payload as { op: string; dataset: string; rows?: string[][]; columns?: unknown[] };
    if (msg.dataset !== "channels") return;

    if (msg.op === "snapshot" && msg.rows) {
      this.channels = msg.rows.map((r) => ({ id: r[0], name: r[1], topic: r[2] }));
      this.render();
      if (this.channels.length > 0 && !this.selectedId) {
        queueMicrotask(() => this.selectChannel(this.channels[0].id));
      }
    } else if (msg.op === "append" && msg.rows) {
      for (const r of msg.rows) {
        this.channels.push({ id: r[0], name: r[1], topic: r[2] });
      }
      this.render();
    }
  };

  private selectChannel(id: string): void {
    this.selectedId = id;
    this.render();
    this.dispatchEvent(new CustomEvent("pages-event", {
      bubbles: true,
      composed: true,
      detail: { topic: "channel-selected", payload: { channelId: id } },
    }));
  }

  private render(): void {
    this.shadow.innerHTML = `
      <style>${STYLES}</style>
      <div class="header">Channels</div>
      <div class="channel-list">
        ${this.channels.map((ch) => `
          <div class="channel ${ch.id === this.selectedId ? "selected" : ""}"
               data-id="${ch.id}"
               title="${ch.topic || ch.name}">
            <span class="channel-hash">#</span>${ch.name}
          </div>
        `).join("")}
      </div>
    `;
    this.shadow.querySelectorAll(".channel").forEach((el) => {
      el.addEventListener("click", () => {
        this.selectChannel((el as HTMLElement).dataset.id!);
      });
    });
  }
}

customElements.define("chat-channel-sidebar", ChatChannelSidebar);
```

- [ ] **Step 2: Build and smoke test**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn quarkus:dev -pl chat-demo -Pdemo -Pui`
Open: `http://localhost:8090`
Expected: Channel sidebar shows seed channels from the database, clicking a channel highlights it, dock toggle hides/shows the panel.

- [ ] **Step 3: Commit**

```
git add chat-demo/src/main/webui/src/
git commit -m "feat(chat-demo): channel sidebar panel with real-time WebSocket data — Refs #28"
```

---

### Task 5: Message List Panel

**Files:**
- Modify: `chat-demo/src/main/webui/src/panels/message-list.ts`

**Interfaces:**
- Consumes: `ws-data` events (dataset `messages`), `channel-selected` events
- Produces: Rendered message list filtered by selected channel, with scroll anchoring and message grouping

- [ ] **Step 1: Implement the message list**

Replace `chat-demo/src/main/webui/src/panels/message-list.ts`. Key features:

- Filters messages by `channelId` matching the selected channel
- Groups consecutive messages from same sender within 2 minutes (collapsed — only first shows sender/time)
- Auto-scrolls to bottom only if user is at the bottom; shows "New messages ↓" pill if scrolled up
- Formats timestamps: `HH:mm` for today, `MMM d, HH:mm` for older
- Empty state: "No messages yet" centered in muted text
- Uses Shadow DOM for style isolation

The implementation should use the same patterns as the channel sidebar:
- Listen for `ws-data` events, filter by `dataset: "messages"`
- On `snapshot`, store all messages; on `append`, add new ones
- Listen for `channel-selected` to update filter and re-render

CSS should ensure:
- Messages have `padding: 8px 16px`, `margin-bottom: 2px`
- Sender name: accent color, bold, 13px
- Timestamp: right-aligned, muted, 11px
- Message text: 14px, `line-height: 1.5`
- Grouped follow-up messages: reduced top padding (2px), no sender/time header
- Scroll container fills available height with `overflow-y: auto`
- "New messages ↓" pill: fixed at bottom of scroll container, accent background, rounded, clickable

- [ ] **Step 2: Build and smoke test**

Run dev mode, open browser. Click channels — messages should filter. Send a message via REST (`curl -X POST -H 'Content-Type: application/json' -d '{"text":"hello from curl"}' http://localhost:8090/api/channels/{id}/messages`) and verify it appears in real-time. Check scroll anchoring by scrolling up, sending another message, and verifying the "New messages" pill appears.

- [ ] **Step 3: Commit**

```
git add chat-demo/src/main/webui/src/panels/message-list.ts
git commit -m "feat(chat-demo): message list panel with grouping and scroll anchoring — Refs #28"
```

---

### Task 6: Compose Input Panel

**Files:**
- Modify: `chat-demo/src/main/webui/src/panels/message-input.ts`

**Interfaces:**
- Consumes: `channel-selected` events (to know which channel to POST to)
- Produces: `POST /api/channels/{channelId}/messages` on Enter key

- [ ] **Step 1: Implement the compose input**

Replace `chat-demo/src/main/webui/src/panels/message-input.ts`. Key features:

- Full-width text input with placeholder "Type a message..."
- Listens for `channel-selected` to store current `channelId`
- On Enter: POST `{"text": "..."}` to `/api/channels/{channelId}/messages`
- Clears input on success
- Disabled with muted placeholder when no channel selected
- Disabled during POST (prevents double-send)
- On POST failure: re-enable, flash border red for 2 seconds
- Styled to match dark theme

CSS should ensure:
- Input fills width with `padding: 10px 16px`
- Background: `var(--pages-bg-alt)`, border: `1px solid var(--pages-border)`
- Border-radius: `6px`, font-size: `14px`
- Focus: accent border color
- Disabled: reduced opacity
- No box-shadow or outline that looks jarring in dark mode

- [ ] **Step 2: Build and smoke test**

Run dev mode, open browser. Type a message and press Enter — message should appear in the message list via WebSocket append. Input should clear. Try submitting with no channel selected — should be disabled.

- [ ] **Step 3: Commit**

```
git add chat-demo/src/main/webui/src/panels/message-input.ts
git commit -m "feat(chat-demo): compose input panel with Enter-to-send — Refs #28"
```

---

### Task 7: Member List Panel

**Files:**
- Modify: `chat-demo/src/main/webui/src/panels/member-list.ts`

**Interfaces:**
- Consumes: `ws-data` events (datasets `members` and `presence`), `channel-selected` events
- Produces: Rendered member list with presence indicators, sorted by status then name

- [ ] **Step 1: Implement the member list**

Replace `chat-demo/src/main/webui/src/panels/member-list.ts`. Key features:

- Dual dataset subscription: `members` for the list, `presence` for status dots
- Maintains local `presenceMap: Map<string, string>` from presence snapshot and replace events
- Filters members by `channelId` matching the selected channel
- Joins member rows with presence map by `memberId`
- Sorts: ONLINE first, AWAY second, OFFLINE/UNKNOWN last; alphabetical within groups
- Presence dot: 10px circle, green (#4caf50) = ONLINE, amber (#ffc107) = AWAY, grey (#757575) = OFFLINE/UNKNOWN
- "Members" header with count badge (small rounded pill showing member count)
- Scrollable list

CSS should ensure:
- Member rows: `padding: 6px 16px`, `font-size: 13px`
- Presence dot: `width: 8px; height: 8px; border-radius: 50%; display: inline-block; margin-right: 8px`
- Count badge: small, muted, rounded
- Display names truncated with ellipsis if panel is narrow

- [ ] **Step 2: Build and smoke test**

Run dev mode, open browser. Members should show with presence dots. Set presence via REST (`curl -X PUT -H 'Content-Type: application/json' -d '{"status":"ONLINE"}' http://localhost:8090/api/presence/alice`) and verify the dot updates in real-time. Click different channels — members should filter.

- [ ] **Step 3: Commit**

```
git add chat-demo/src/main/webui/src/panels/member-list.ts
git commit -m "feat(chat-demo): member list panel with presence indicators — Refs #28"
```

---

### Task 8: UX Polish and Final Integration

**Files:**
- Potentially all panel files and index.ts — refinements only

**Interfaces:**
- Consumes: All previous task outputs
- Produces: Polished, showcase-quality UI

- [ ] **Step 1: Run full application and review UX holistically**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn quarkus:dev -pl chat-demo -Pdemo -Pui`

Check each of these systematically in the browser:

1. **No clipped text** — resize browser window to minimum, verify no text is cut off without ellipsis
2. **Panel resize** — drag split handles, verify panels resize smoothly and content reflows
3. **Dock toggle** — close/open both side panels, verify message area fills the space
4. **Dark theme consistency** — no white backgrounds, no jarring contrast, all panels use CSS vars
5. **Scroll behavior** — fill a channel with many messages (via REST), verify scroll anchoring works
6. **Channel switching** — click through channels, verify messages/members filter instantly
7. **Empty states** — create a new channel via REST, select it, verify "No messages yet" shows
8. **Input focus** — click in compose input, verify focus ring is visible and attractive
9. **Typography** — verify font sizes are readable, line heights are comfortable, spacing is consistent
10. **Presence updates** — change presence via REST, verify dot color changes in real-time

- [ ] **Step 2: Fix any issues found**

Address any visual regressions, alignment issues, or UX rough edges found in Step 1. Common fixes:
- Add `min-height` to prevent panels from collapsing to zero
- Adjust split ratios or min sizes
- Fix font smoothing for dark backgrounds (`-webkit-font-smoothing: antialiased`)
- Ensure consistent padding between panels
- Verify the compose input height is comfortable (not too small, not too tall)

- [ ] **Step 3: Run full build**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn clean install -Pdemo -Pui`
Expected: BUILD SUCCESS

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn clean install -Pdemo`
Expected: BUILD SUCCESS (no Node.js required)

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn clean install`
Expected: BUILD SUCCESS (chat-demo module not built)

- [ ] **Step 4: Commit**

```
git add -A
git commit -m "feat(chat-demo): UX polish pass — Refs #28"
```
