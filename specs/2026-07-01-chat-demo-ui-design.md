# Chat Demo UI — casehub-pages Quinoa Frontend

**Issue:** casehubio/connectors#28
**Date:** 2026-07-01
**Status:** Design

## Context

The chat-demo module has a complete REST + WebSocket backend (`ChatResource`, `ChatWebSocket`, `ChatWebSocketBroadcaster`, `SqliteChatBackend`) with SQLite persistence and a pre-populated seed database. This spec covers adding a casehub-pages frontend via Quinoa, served alongside the Quarkus backend.

Blockers resolved: casehubio/casehub-pages#52 (WebSocket dataset provider), #53 (multiplexing).

Note: casehub-pages#54 (submit action on form inputs) is **not** a blocker for this spec. The pages `SubmitConfig.url` is a static `readonly string` set at construction time, but the compose input requires a dynamic URL (`/api/channels/{selectedChannelId}/messages`) that changes at runtime based on channel selection. The compose input uses a custom Web Component with manual fetch logic instead.

## Layout

Three-column resizable workspace using casehub-pages workbench primitives (`split`, `dockBar`, `hostPanel`). Dark mode default.

```
┌──┬──────────────┬────────────────────────────────┬───────────────┐
│  │  Channels    │         Messages               │   Members     │
│💬│              │                                │               │
│  │  #general ◀  │  alice  10:32am                │  ● alice      │
│👥│  #random     │  Hey everyone!                 │  ● bob        │
│  │  #design     │                                │  ○ charlie    │
│  │              │  bob    10:33am                 │               │
│  │              │  Morning! How's the sprint?     │               │
│  │              │                                │               │
│  │              │  ┌──────────────────────────┐  │               │
│  │              │  │ Type a message...    [⏎] │  │               │
│  │              │  └──────────────────────────┘  │               │
└──┴──────────────┴────────────────────────────────┴───────────────┘
```

- **Dock bar** on left edge with two icons: 💬 (channels, default open), 👥 (members, default open)
- **Split ratio:** 20 / 60 / 20 with min sizes to prevent unusable collapse
- **Message area:** vertical split — scrollable message list (90%) + fixed compose input (10%)
- Collapsing a side panel redistributes space to the message area

### Component Tree (TypeScript DSL)

```typescript
columns([0, 1],
  [dockBar("vertical", [
    { icon: "💬", label: "Channels", panelId: "channel-panel", defaultOpen: true },
    { icon: "👥", label: "Members", panelId: "member-panel", defaultOpen: true },
  ])],
  [split("horizontal", [
    withId("channel-panel", hostPanel("channels")),
    split("vertical", [
      hostPanel("messages"),
      hostPanel("input"),
    ], { ratio: [90, 10] }),
    withId("member-panel", hostPanel("members")),
  ], { ratio: [20, 60, 20] })],
)
```

## Data Architecture

### Single Multiplexed WebSocket

All datasets flow over `ws://localhost:8090/ws/chat`. On connect, the server sends a snapshot array; subsequent mutations arrive as individual ops.

### Wire Protocol (pages-compatible)

The existing `ChatWebSocketBroadcaster` must be updated to match the casehub-pages wire protocol:

| Change | Before | After |
|--------|--------|-------|
| Op field | `type` | `op` |
| Schema | absent | `columns` array on snapshot/append/replace |
| Ordering | absent | monotonic `seq` counter |

All row values are strings or null — no JSON booleans, numbers, or objects. The `Column` type requires `name` alongside `id` and `type`.

Message examples:

```json
{"op":"snapshot","dataset":"channels","seq":"1",
 "columns":[{"id":"id","name":"ID","type":"LABEL"},{"id":"name","name":"Name","type":"LABEL"},{"id":"topic","name":"Topic","type":"LABEL"},{"id":"description","name":"Description","type":"LABEL"},{"id":"isPrivate","name":"Private","type":"LABEL"}],
 "rows":[["ch-1","general","General chat","","false"]]}

{"op":"append","dataset":"messages","seq":"12",
 "columns":[{"id":"channelId","name":"Channel","type":"LABEL"},{"id":"messageId","name":"Message ID","type":"LABEL"},{"id":"parentId","name":"Parent","type":"LABEL"},{"id":"senderId","name":"Sender","type":"LABEL"},{"id":"text","name":"Text","type":"LABEL"},{"id":"timestamp","name":"Timestamp","type":"DATE"}],
 "rows":[["ch-1","msg-42","","alice","Hey everyone!","2026-07-01T10:32:00Z"]]}

{"op":"replace","dataset":"presence","seq":"15",
 "columns":[{"id":"memberId","name":"Member","type":"LABEL"},{"id":"status","name":"Status","type":"LABEL"}],
 "row":["alice","ONLINE"],"key":"alice"}

{"op":"remove","dataset":"members","seq":"16","key":"general:bob"}
```

### Datasets

| Dataset | Snapshot | Live ops | Key column | Consumers |
|---------|----------|----------|------------|-----------|
| `channels` | All channels | `append` on create | — | Channel sidebar |
| `messages` | All messages (all channels) | `append` on send/reply | — | Message list (client-filtered by selected channel) |
| `members` | All memberships (with composite `membershipId`) | `append`/`remove` | `membershipId` | Member panel (client-filtered by selected channel) |
| `presence` | All member presence states | `replace` on status change | `memberId` | Member panel (presence dots) |
| `reactions` | — | `append` | — | No UI consumer this iteration (backend broadcasts, frontend ignores) |

`keyColumn` is required for `replace` and `remove` operations — without it, the pages data layer silently drops those events (push-source.ts:119-122).

### TypeScript Dataset Definitions

The frontend registers datasets using the `dataset()` builder from `pages-ui/src/dsl/builders.ts:422`, which produces valid `ExternalDataSetDef` objects (mapping `id` → `uuid`, including `url`). Each definition's URL includes `?dataset=<name>` so `extractWireName` (websocket-source.ts:36) can route incoming wire messages to the correct subscription.

The `createWebSocketSource` is called with the base URL; individual dataset definitions provide their URLs for wire name extraction:

```typescript
const WS_BASE = "ws://localhost:8090/ws/chat";

const datasets = [
  dataset("channels", `${WS_BASE}?dataset=channels`, {
    columns: [
      { id: "id", name: "ID", type: ColumnType.LABEL },
      { id: "name", name: "Name", type: ColumnType.LABEL },
      { id: "topic", name: "Topic", type: ColumnType.LABEL },
      { id: "description", name: "Description", type: ColumnType.LABEL },
      { id: "isPrivate", name: "Private", type: ColumnType.LABEL },
    ],
  }),
  dataset("messages", `${WS_BASE}?dataset=messages`, {
    columns: [
      { id: "channelId", name: "Channel", type: ColumnType.LABEL },
      { id: "messageId", name: "Message ID", type: ColumnType.LABEL },
      { id: "parentId", name: "Parent", type: ColumnType.LABEL },
      { id: "senderId", name: "Sender", type: ColumnType.LABEL },
      { id: "text", name: "Text", type: ColumnType.LABEL },
      { id: "timestamp", name: "Timestamp", type: ColumnType.DATE },
    ],
  }),
  dataset("members", `${WS_BASE}?dataset=members`, {
    keyColumn: "membershipId",
    columns: [
      { id: "membershipId", name: "Membership", type: ColumnType.LABEL },
      { id: "channelId", name: "Channel", type: ColumnType.LABEL },
      { id: "memberId", name: "Member", type: ColumnType.LABEL },
      { id: "displayName", name: "Display Name", type: ColumnType.LABEL },
    ],
  }),
  dataset("presence", `${WS_BASE}?dataset=presence`, {
    keyColumn: "memberId",
    columns: [
      { id: "memberId", name: "Member", type: ColumnType.LABEL },
      { id: "status", name: "Status", type: ColumnType.LABEL },
    ],
  }),
];

const source = createWebSocketSource(WS_BASE);
```

The `reactions` dataset is omitted — no UI consumer exists this iteration (casehubio/connectors#50), so reaction events are silently dropped by the multiplexer.

### Inter-Panel Communication

Channel selection uses `pages-event` DOM events. All consuming panels — message list, member panel, **and compose input** — listen for `channel-selected`:

```typescript
// Sidebar dispatches
this.dispatchEvent(new CustomEvent("pages-event", {
  bubbles: true, composed: true,
  detail: { topic: "channel-selected", payload: { channelId: "ch-1" } },
}));

// Message list, member panel, and compose input listen
document.addEventListener("pages-event", (e) => {
  const { topic, payload } = (e as CustomEvent).detail;
  if (topic === "channel-selected") {
    this.setChannel(payload.channelId);
  }
});
```

No server round-trip for channel switching — panels filter locally from the full dataset.

**Initialization ordering:** The sidebar's auto-selection on snapshot load (first channel) must not fire before other panels have registered their `pages-event` listeners. The sidebar defers its initial `channel-selected` dispatch via `queueMicrotask()` — all Web Components from the same `loadSite()` call are synchronously constructed and connected before microtasks run, guaranteeing listeners are registered before the first event fires.

## Panel Designs

### Channel Sidebar (`chat-channel-sidebar`)

- Each row: `# channel-name`, truncated with `text-overflow: ellipsis` — never wraps
- Selected channel: highlighted background (`--pages-bg-hover`)
- Click dispatches `channel-selected` event
- First channel auto-selected on snapshot load
- Subtle "Channels" header in muted text
- Scrollable with hidden scrollbar styling
- Spacing: `padding: 8px 12px` per item, `font-size: 14px`

### Message List (`chat-message-list`)

- Listens for `channel-selected`, filters messages locally
- Message block layout:
  - **Sender** in accent color, bold, 13px
  - **Timestamp** right-aligned, muted, 11px — `HH:mm` for today, `MMM d, HH:mm` for older
  - **Text** below sender line, 14px, `line-height: 1.5`
  - Spacing: `padding: 8px 16px`, `margin-bottom: 4px`
- **Message grouping:** Consecutive messages from same sender within 2 minutes collapse — sender/time shown only on first, subsequent messages show text only with reduced top padding
- **Scroll anchoring:** Auto-scroll to bottom on new messages only if user is already at bottom. If scrolled up, show a "New messages ↓" pill that scrolls down on click
- **Empty state:** "No messages yet" centered in muted text

### Compose Input (`chat-message-input`)

- Full-width text input, placeholder "Type a message..."
- Styled with theme CSS vars: `--pages-bg-alt` background, `--pages-text` color, `--pages-border` border
- Listens for `channel-selected` events and stores the current `channelId`
- Enter sends message via fetch:
  ```
  POST /api/channels/{channelId}/messages
  Content-Type: application/json
  {"text": "message content"}
  ```
  `ChatResource.postMessage()` expects `PostMessageRequest(String text)`.
- Clears on success, disabled during request (prevents double-send)
- On POST failure: re-enable input, flash border red (`--pages-error` or `#ef5350`) for 2 seconds
- Disabled with muted placeholder when no channel is selected
- Single line — no multiline support

### Member Panel (`chat-member-list`)

This is the only panel that consumes two datasets: `members` (for the member list) and `presence` (for status dots).

- Subscribes to the `members` dataset — filters by `channelId` matching the selected channel
- Subscribes to the `presence` dataset — maintains a local `memberId → status` map, updated on `replace` events. The presence snapshot on connect provides initial state so members display correct status immediately (no grey-then-update flicker)
- Listens for `channel-selected` to update the channel filter
- Member row: presence dot (10px circle) + display name
  - Green `#4caf50` = ONLINE
  - Amber `#ffc107` = AWAY
  - Grey `#757575` = OFFLINE/UNKNOWN
- On render: joins member rows with the presence map by `memberId` to determine each member's dot color
- Sorted: ONLINE first, then AWAY, then OFFLINE; alphabetical within groups
- "Members" header with count badge
- Scrollable, `padding: 6px 12px` per row, `font-size: 13px`

## Build Integration

### File Layout

```
chat-demo/
  pom.xml                    (updated: Quinoa dependency, -Pui profile)
  src/main/
    java/...                  (existing backend + broadcaster updates)
    resources/
      application.properties  (updated: Quinoa config, inert without -Pui)
    webui/
      package.json            (file: deps to local pages packages)
      esbuild.config.mjs      (from quinoa-host template)
      tsconfig.json            (from quinoa-host template)
      .gitignore               (node_modules/, dist/)
      src/
        index.ts               (loadSite entry point)
        index.html             (shell HTML)
        panels/
          channel-sidebar.ts   (Web Component)
          message-list.ts      (Web Component)
          message-input.ts     (Web Component)
          member-list.ts       (Web Component)
```

### Profile Gating

| Command | chat-demo | UI | Node.js needed |
|---------|-----------|----|----|
| `mvn clean install` | No | No | No |
| `mvn clean install -Pdemo` | Yes | No | No |
| `mvn clean install -Pdemo -Pui` | Yes | Yes | Yes (auto-installed by Quinoa) |
| `mvn quarkus:dev -Pdemo -Pui` | Yes | Yes + hot reload | Yes |

Quinoa dependency lives inside the `-Pui` Maven profile. When `-Pui` is not active, Quinoa is not on the classpath — no Node.js download, no npm build, no webui processing. When active, Quinoa detects `src/main/webui/` and handles everything.

### Quinoa Configuration

```properties
# Only active when Quinoa is on the classpath (-Pui profile)
quarkus.quinoa.build-dir=dist
quarkus.quinoa.package-manager-install=true
quarkus.quinoa.package-manager-install.node-version=22.16.0
quarkus.quinoa.enable-spa-routing=true
```

SPA routing catches unmatched paths and returns `index.html`. Quinoa integrates with Quarkus's routing layer — routes handled by JAX-RS (`/api/*`) and WebSocket (`/ws/chat`) take priority. SPA routing only applies to paths not matched by any Quarkus route.

### Maven Profile

```xml
<profile>
  <id>ui</id>
  <dependencies>
    <dependency>
      <groupId>io.quarkiverse.quinoa</groupId>
      <artifactId>quarkus-quinoa</artifactId>
    </dependency>
  </dependencies>
</profile>
```

### package.json Dependencies

```json
{
  "dependencies": {
    "@casehubio/pages-runtime": "file:../../../../casehub/pages/packages/pages-runtime",
    "@casehubio/pages-ui": "file:../../../../casehub/pages/packages/pages-ui"
  }
}
```

Relative path from `chat-demo/src/main/webui/` to the local pages repo. Adjust if your clone layout differs.

## Backend Changes

### ChatWebSocketBroadcaster

- Rename `type` → `op` in all broadcast messages
- Add `columns` array to `snapshot`, `append`, and `replace` messages
- Add `AtomicLong seq` counter, increment per broadcast, include as string `seq` field
- Extract column definitions as constants (one per dataset)
- Serialise all row values as strings — `String.valueOf(ch.isPrivate())` instead of bare boolean (the wire protocol requires `(string | null)[][]`)
- Add `membershipId` composite column to members dataset: `channelId + ":" + memberId` as first column in all member rows (snapshot, append, remove key)
- Add presence snapshot: collect unique `MemberRef` instances across all channels (deduplicate by `memberId` — a member in multiple channels should produce one presence row, not one per channel), query `chatPlatform.presence().of(memberRef)` for each, build `[memberId, status.name()]` rows
- Snapshot on connect sends four datasets: channels, messages, members, presence

### ChatWebSocket

- Add `@OnTextMessage` handler. The pages WebSocket client sends `{"op":"subscribe","dataset":"..."}` on connect and `{"op":"unsubscribe","dataset":"..."}` on disconnect. The demo server broadcasts everything regardless of subscriptions, so these messages are logged at DEBUG level and not acted upon. The handler prevents silent message drops and provides a clear extension point if per-dataset subscription is added later.

### Sender Identity

All messages sent via the compose input go through `ChatResource.postMessage()` → `chatPlatform.messaging().send()`. In `RefChatPlatform`, the sender is always `new MemberRef(id())` where `id()` returns `"ref"`. Every outbound message has `senderId = "ref"` in the messages dataset. The UI should render this as the platform's own identity (e.g., display "You" or use the `"ref"` string directly).

This is intentional for the demo — the `ChatResource` calls the outbound messaging path directly rather than the L2 inbound event path described in the prior demo spec. The inbound path (`InboundMessage → RefInboundTranslator → ReceivedMessage`) would allow per-user sender identity, but is not exercised by the demo module. User identity is out of scope for this iteration.

### No Other Backend Changes

REST API, WebSocket endpoint path, SQLite schema, seed database — all unchanged.

## Testing

### REST API Tests

No changes needed to `ChatResourceTest`. REST responses from `ChatResource` return domain objects (`Channel`, `ReceivedMessage`, `Map`) serialised by Jackson — these are unaffected by wire protocol changes. The wire protocol (`op`, `columns`, `seq`) is WebSocket-only, covered by `ChatWebSocketTest` below.

### WebSocket Integration Tests

New `ChatWebSocketTest` class using Quarkus WebSocket test client:

- **Snapshot structure:** Connect, verify snapshot contains four datasets (channels, messages, members, presence) each with `op`, `columns`, `rows`, `seq`
- **Column compliance:** Verify every column object has `id`, `name`, and `type` fields
- **Row type safety:** Verify all row values are strings or null — no booleans, numbers, or objects
- **Append event:** POST a message via REST, verify the append event arrives on the WebSocket with correct structure and `seq` greater than the snapshot's
- **Seq monotonicity:** Verify `seq` values are monotonically increasing across all events in a connection
- **MembershipId:** Verify members rows include `membershipId` as first column with value `channelId:memberId`
- **Presence snapshot:** Verify presence dataset has snapshot rows with `[memberId, status]` structure
- **Presence deduplication:** Verify a member in multiple channels produces exactly one presence row in the snapshot

### Manual UI Testing

- UI not testable in `@QuarkusTest` (Quinoa is disabled — GE-20260701-c000c7). Manual verification via `mvn quarkus:dev -Pdemo -Pui`.

## Garden Context

| Entry | Relevance |
|-------|-----------|
| GE-20260630-bf0055 | Quinoa requires explicit `node-version` — pinned in config |
| GE-20260630-d5cad9 | Quinoa extension version must match Maven Central — verified |
| GE-20260701-c000c7 | `@QuarkusTest` disables Quinoa — UI needs manual testing |
| GE-20260629-59c7e6 | esbuild minifies constants in template literals — avoid that pattern |
| GE-20260701-f1c67f | IntelliJ MCP doesn't index TypeScript — use bash for TS |

## Out of Scope

Each deferred item has a corresponding GitHub issue for tracking:

- Thread view / reply UI — threading exists in the API but not surfaced in this iteration (casehubio/connectors#49)
- Reaction UI — backend broadcasts reactions but no UI to add/view them (casehubio/connectors#50)
- Channel creation UI — use REST API directly (casehubio/connectors#51)
- User identity / login — the demo uses `"ref"` as sender for all messages (casehubio/connectors#52)
