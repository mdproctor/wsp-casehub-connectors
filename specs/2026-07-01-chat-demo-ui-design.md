# Chat Demo UI — casehub-pages Quinoa Frontend

**Issue:** casehubio/connectors#28
**Date:** 2026-07-01
**Status:** Design

## Context

The chat-demo module has a complete REST + WebSocket backend (`ChatResource`, `ChatWebSocket`, `ChatWebSocketBroadcaster`, `SqliteChatBackend`) with SQLite persistence and a pre-populated seed database. This spec covers adding a casehub-pages frontend via Quinoa, served alongside the Quarkus backend.

Blockers resolved: casehubio/casehub-pages#52 (WebSocket dataset provider), #53 (multiplexing), #54 (submit action on form inputs).

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
| Schema | absent | `columns` array on snapshot/append |
| Ordering | absent | monotonic `seq` counter |

Message examples:

```json
{"op":"snapshot","dataset":"channels","seq":"1",
 "columns":[{"id":"id","type":"LABEL"},{"id":"name","type":"LABEL"},{"id":"topic","type":"LABEL"},{"id":"description","type":"LABEL"},{"id":"isPrivate","type":"LABEL"}],
 "rows":[["ch-1","general","General chat","",false]]}

{"op":"append","dataset":"messages","seq":"12",
 "columns":[{"id":"channelId","type":"LABEL"},{"id":"messageId","type":"LABEL"},{"id":"parentId","type":"LABEL"},{"id":"senderId","type":"LABEL"},{"id":"text","type":"LABEL"},{"id":"timestamp","type":"LABEL"}],
 "rows":[["ch-1","msg-42","","alice","Hey everyone!","2026-07-01T10:32:00Z"]]}

{"op":"replace","dataset":"presence","seq":"15",
 "columns":[{"id":"memberId","type":"LABEL"},{"id":"status","type":"LABEL"}],
 "row":["alice","ONLINE"],"key":"alice"}

{"op":"remove","dataset":"members","seq":"16","key":"general:bob"}
```

### Datasets

| Dataset | Snapshot | Live ops | Consumers |
|---------|----------|----------|-----------|
| `channels` | All channels | `append` on create | Channel sidebar |
| `messages` | All messages (all channels) | `append` on send/reply | Message list (client-filtered by selected channel) |
| `members` | All memberships | `append`/`remove` | Member panel (client-filtered by selected channel) |
| `presence` | — | `replace` on status change | Member panel (presence dots) |
| `reactions` | — | `append` | Message list (reaction badges) |

### Inter-Panel Communication

Channel selection uses `pages-event` DOM events:

```typescript
// Sidebar dispatches
this.dispatchEvent(new CustomEvent("pages-event", {
  bubbles: true, composed: true,
  detail: { topic: "channel-selected", payload: { channelId: "ch-1" } },
}));

// Message list and member panel listen
document.addEventListener("pages-event", (e) => {
  const { topic, payload } = (e as CustomEvent).detail;
  if (topic === "channel-selected") {
    this.setChannel(payload.channelId);
  }
});
```

No server round-trip for channel switching — panels filter locally from the full dataset.

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
- Enter sends: `POST /api/channels/{channelId}/messages` via fetch
- Clears on success, disabled during request (prevents double-send)
- Single line — no multiline support

### Member Panel (`chat-member-list`)

- Listens for `channel-selected`, filters members locally
- Member row: presence dot (10px circle) + display name
  - Green `#4caf50` = ONLINE
  - Amber `#ffc107` = AWAY
  - Grey `#757575` = OFFLINE/UNKNOWN
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
- Add `columns` array to `snapshot` and `append` messages
- Add `AtomicLong seq` counter, increment per broadcast, include as string `seq` field
- Extract column definitions as constants (one per dataset)
- Snapshot on connect already sends all three datasets — add column metadata

### No Other Backend Changes

REST API, WebSocket endpoint path, SQLite schema, seed database — all unchanged.

## Testing

- Update existing `ChatResourceTest` assertions for wire protocol changes (`op`, `columns`, `seq`)
- New unit test: verify snapshot structure matches pages wire protocol (field names, column types, row shapes)
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

- Thread view / reply UI (threading exists in the API but not surfaced in this iteration)
- Reaction UI (backend broadcasts reactions but no UI to add/view them)
- Channel creation UI (use REST API directly)
- User identity / login (the demo uses a hardcoded sender)
