# Qhorus Chat UI — Design Spec

**Date:** 2026-07-07
**Status:** Draft
**Repo:** casehubio/connectors (chat-demo module)
**Depends on:** casehubio/qhorus#328 (model enrichments — phases 3–4)

## Problem

The chat-demo webui is a Web Components app (4 panel components + 3
infrastructure modules, ~60KB production TypeScript) built on pages-runtime
and pages-ui for layout (`loadSite`, `split`, `dockBar`, `hostPanel`), panel
hosting (`registerPanel`), identity (`PagesDevAuth`, `PagesIdentity`), and
event routing (`pages-event` custom events). However, individual components
extend raw `HTMLElement` with Shadow DOM and manual `innerHTML` rendering —
they don't use Lit, the pages dataset pipeline, or accessibility
infrastructure. A hand-built `ResponsiveController` handles viewport
adaptation outside the pages layout system.

The result works but is hard to embed in other applications (components are
tightly coupled to the WebSocket lifecycle in `index.ts`), lacks
accessibility, and can only render flat chat messages.

Meanwhile, qhorus channels carry rich structured conversations — speech acts,
correlation chains, normative layers, commitment lifecycles — that the current
UI flattens into a Slack-like text stream.

## Vision

A component family that renders qhorus channel conversations with full
structural awareness: speech act badges, correlation chain grouping, normative
layer navigation, topic-based sub-conversations, dockable contextual panels
for artifacts/tasks/correlation. Built on pages/blocks conventions (Lit,
pages dataset pipeline, design tokens, split/dockBar layout). Embeddable as a whole in
any pages application (claudony, case workbenches) or decomposed — mount just
the feed, just the channel nav, just a single message renderer.

The UI is qhorus-native but backward-compatible with simpler chat backends.
A Slack message renders as a simple chat message. A qhorus COMMAND→STATUS→DONE
chain renders as a collapsible obligation lifecycle with commitment state
badges.

## Non-Goals

- Replacing Slack/Teams/Discord for enterprise human chat
- Building a standalone chat application (this is always embedded)
- Implementing qhorus model changes (filed separately as qhorus#328)
- Drafthouse-specific rendering (drafthouse has its own projections)

---

## 1. Conversation Model

### Hierarchy

**Space → Channel → Topic → Thread**

| Level | Definition | Qhorus mapping |
|-------|-----------|----------------|
| **Space** | Organizational container for related channels. A case, a project, or a team. | New concept (qhorus#328). Channels gain optional `spaceId`. Normative triples auto-create a Space. |
| **Channel** | Conversation container with semantics and access control. The five channel semantics (APPEND, COLLECT, BARRIER, EPHEMERAL, LAST_WRITE) and type constraints (allowedTypes/deniedTypes) are unchanged. | Existing qhorus concept — unchanged. |
| **Topic** | Named, persistent sub-conversation within a channel. Inspired by Zulip's mandatory topic model. Every message belongs to a topic; messages without explicit topic go to "General" default. | New concept (qhorus#328). `topic` string field on Message. |
| **Thread** | Correlation chain within a topic. A COMMAND and its STATUS/DONE responses. A QUERY and its RESPONSE. Already implicit via `correlationId` and `inReplyTo`. | No model change — UI renders existing fields as visual grouping. |

### Why this hierarchy

Research across 11 chat platforms and 6 agent communication frameworks
(see [conversation-model-research.md](2026-07-07-conversation-model-research.md))
identified three key findings:

1. **Optional threading degrades to no threading** (Zulip's central insight).
   For agent communication, mandatory topics solve the "5 agents, 5 tasks,
   1 channel" readability problem.
2. **Named sub-conversations > anonymous reply chains** for navigability,
   resumability, and human comprehension of agent work.
3. **Recursive container nesting** (Matrix spaces) provides flexibility
   without special-casing hierarchy types.

### Backward compatibility with chat platforms

All enrichments are additive. Existing ChatPlatform SPI integrations
(Slack, IRC, Discord, Teams, Telegram, Signal) continue unchanged:

| Source | Space | Channel | Topic | Thread |
|--------|-------|---------|-------|--------|
| Slack | — | maps to qhorus channel | "General" (default) | Slack thread ts → parentRef (platform-native ref; `inReplyTo` is null in v1 — see note below) |
| IRC | — | maps to qhorus channel | "General" | — |
| Teams | — | maps to qhorus channel | "General" | Teams reply → inReplyTo |
| Qhorus agent | Case space | work/observe/oversight | Named per task | correlationId chain |
| Drafthouse | Session space | debate/review channel | Named per review point | Entry chain |

**Slack thread note:** The `SlackInboundTranslator` maps Slack `thread_ts`
to `parentRef` — a `ChatMessageRef` containing the platform-native thread
timestamp. This is distinct from `inReplyTo`, which carries a qhorus ledger
ID and is always null in v1 (the gateway's `receiveHumanMessage()` is void,
so there is no way to build a `slackTs → ledgerId` mapping). The UI adapter
can use `parentRef` for visual thread grouping in the feed, but correlation
chain features (§2 `<qhorus-thread>` commitmentState, correlation panel)
require `inReplyTo`/`correlationId` which are only available for
qhorus-native messages.

### Five conversation patterns

| Pattern | Space | Channels | Topics | UI mode |
|---------|-------|----------|--------|---------|
| Simple chat (Slack-like) | Optional | One channel | Default "General" | Flat stream |
| Normative triple | Case space | work + observe + oversight | One per task/obligation | Grouped triple view |
| Drafthouse debate | Session space | debate + review | One per review point | Debate structure |
| Ad-hoc group DM | — | One private channel | Default | Flat stream |
| Case investigation | Case space | work channel | Multiple (one per investigation) | Topic navigator |

---

## 2. Component Architecture

Three layers, each independently usable.

### Primitives (LitElement — pure rendering, props-driven)

#### `<qhorus-message>`
Renders a single message. Receives message data as properties.

**Renders:**
- Speech act badge (pill: QUERY, COMMAND, RESPONSE, STATUS, DONE, FAILURE,
  DECLINE, HANDOFF, EVENT) — color-coded per §8 mapping:
  - Information exchange (QUERY, RESPONSE, STATUS) → `--pages-info-*`
  - Obligation initiation (COMMAND) → `--pages-accent-*`
  - Terminal success (DONE) → `--pages-success-*`
  - Terminal failure (FAILURE) → `--pages-danger-*`
  - Terminal refusal (DECLINE) → `--pages-warning-*`
  - Obligation transfer (HANDOFF) → `--pages-info-*`
  - Telemetry (EVENT) → `--pages-neutral-*`
- Actor icon (human silhouette / robot / gear for HUMAN/AGENT/SYSTEM)
- Sender name + timestamp
- Message content (markdown-rendered)
- Reaction bar (if reactions present)
- Artefact chips (clickable, typed — document/code/case/work-item icons)
- Correlation context (expandable — "In reply to..." or "Responding to
  COMMAND: ...")
- Commitment state badge (for COMMAND messages: OPEN/ACKNOWLEDGED/FULFILLED/
  FAILED/DECLINED/DELEGATED/EXPIRED)
- Delegation indicator (for HANDOFF messages: source → target agent transfer)

**Expanded view (click-to-expand / progressive disclosure):**
- Full correlation chain (what this message replies to, what replied to it)
- Artefact details (type, label, selection scope preview)
- Commitment details (deadline, acknowledgedAt, delegation target for
  DELEGATED state, linked work item)
- Topic name and channel

**Properties:**
```typescript
@property() message: QhorusMessage
@property() reactions: Reaction[]
@property() showSpeechAct: boolean = true  // hide in simple chat mode
@property() showActorBadge: boolean = true
@property() expanded: boolean = false
```

#### `<qhorus-message-input>`
Compose area for sending messages.

**Features:**
- Auto-expanding textarea (max height configurable)
- Reply banner (when replying to a specific message)
- Topic selector (current topic name, option to create new topic).
  Requires `topic` field from qhorus#328; hidden until available.
- Speech act selector (dropdown — hidden by default, shown in power-user
  or agent mode). Defaults to EVENT for simple chat.
- Attachment/anchor controls (reference artifacts)
- Send on Enter, newline on Shift+Enter

**Events emitted:**
- `chat:send-message` — `{channelId, content, topic, inReplyTo?, speechAct?, artefactRefs?}`

#### `<qhorus-thread>`
Collapsible group of correlated messages.

**Renders:**
- Root message (the COMMAND, QUERY, or first message in the chain)
- Collapse/expand toggle with summary ("3 status updates, completed")
- Nested `<qhorus-message>` for each message in the chain
- Commitment state on the group header (for obligation chains)
- Thread age/activity indicator
- Delegation fork point (for HANDOFF: visual split showing obligation
  transfer to new agent, with link to child correlation chain)

**Properties:**
```typescript
@property() rootMessage: QhorusMessage
@property() replies: QhorusMessage[]
@property() collapsed: boolean = true
@property() commitmentState?: CommitmentState
```

#### `<qhorus-reaction-bar>`
Emoji reaction pills with add button.

**Renders:**
- Reaction pills: emoji + count + "you reacted" highlight
- Add reaction button (opens emoji palette)
- Click to toggle own reaction

**Events emitted:**
- `chat:react` — `{messageId, emoji}`
- `chat:unreact` — `{messageId, emoji}`

### Composites (LitElement + pages dataset pipeline)

#### `<qhorus-channel-feed>`
Scrollable message stream. The core rendering engine.

**Receives (via pages dataset):**
- `messages` dataset — channel messages with all fields
- `reactions` dataset — reactions keyed by messageId
- `commitments` dataset — commitment states for COMMAND messages

**Features:**
- **Flat mode:** Messages in chronological order, grouped by sender/time
  (messages from same sender within 2 minutes grouped under one header)
- **Threaded mode:** Messages grouped by correlationId into `<qhorus-thread>`
  components. Ungrouped messages render as standalone.
- **Topic filter:** When a topic is selected, shows only messages in that
  topic. Topic headers separate groups.
- **Topic navigator bar** (top of feed): horizontal scrollable list of
  active topics in this channel. Click to filter. "All" to show everything.
  Requires `topic` field from qhorus#328; hidden until available.
- Scroll-to-new-messages pill when scrolled up
- Message selection (click to select → triggers panel updates)
- `prefers-reduced-motion` respected for all animations

**Mode toggle:** Flat/Threaded/Topics — three view modes controlled by
toolbar buttons. Default depends on channel: normative channels default to
Topics, simple chat channels default to Flat. Topic mode requires the
`topic` field on messages (qhorus#328); until then, the toggle shows
Flat and Threaded only.

**Events emitted:**
- `chat:select-topic` — `{channelId, topic}` (when filtering by topic)
- `chat:message-selected` — `{message}` (for panel updates)

#### `<qhorus-channel-nav>`
Channel tree navigator.

**Receives (via pages dataset):**
- `channels` dataset — all accessible channels
- `spaces` dataset — space hierarchy (when available)

**Renders:**
- **Space grouping:** Channels within a space render as expandable tree
  nodes. Normative triples (work/observe/oversight) grouped under their
  case space with combined unread count.
- **Standalone channels:** Channels without a space render at the root level.
- **Unread indicators:** Dot for unread, count badge for mentions.
- **Channel type icons:** Distinct icons for channel semantics (APPEND,
  COLLECT, BARRIER, etc.)
- **Channel actions:** Create channel, delete/archive channel (with
  confirmation dialog using blocks-ui `<blocks-confirm-dialog>`)

**Events emitted:**
- `chat:select-channel` — `{channelId}`
- `chat:create-channel` — `{name, description, spaceId?, semantic?}`
- `chat:delete-channel` — `{channelId}`

#### `<qhorus-member-panel>`
Member list with presence.

**Receives (via pages dataset):**
- `members` dataset — channel membership
- `presence` dataset — member presence states

**Renders:**
- Members sorted: online first, then away, then offline. Alphabetical
  within each group.
- Presence dot (green/yellow/gray/hollow for online/available-busy/away/
  offline)
- Actor type badge (human/agent/system icon)
- Member role indicator (moderator badge)
- Status message if present ("Investigating case-456")

#### `<qhorus-artifact-panel>`
Document/artifact side viewer. Dockable panel.

**Receives:**
- Artefact reference from selected message or clicked chip
- Loads content on demand via configurable resolver

**Renders:**
- Markdown-rendered documents (specs, design docs)
- Syntax-highlighted code with line numbers
- Selection scope highlighting (when ArtefactRef includes selectionScope)
- Navigation: back/forward through viewed artifacts
- "Open in new tab" for full-size viewing

#### `<qhorus-task-panel>`
Commitment/obligation tracker. Dockable panel.

**Receives (via pages dataset):**
- `commitments` dataset — all obligations in the channel

**Renders:**
- List of all COMMAND messages with their commitment state
- State badge: OPEN (blue), ACKNOWLEDGED (teal), FULFILLED (green),
  FAILED (red), DECLINED (gray), DELEGATED (amber), EXPIRED (orange)
- Click to scroll to the originating COMMAND in the feed
- Deadline indicator (overdue highlighted)
- Work item link (when available — future integration)

#### `<qhorus-correlation-panel>`
Full correlation chain viewer. Dockable panel.

**Receives:**
- Selected message from feed

**Renders:**
- Vertical flow diagram: each node is a message in the correlation chain
  (COMMAND → STATUS → STATUS → DONE)
- Nodes are clickable — scroll-to in the feed
- Actor badges on each node
- Timing information (duration between steps)
- Branching for HANDOFF (obligation transferred to different agent)

### Workbench (LitElement + pages layout — full experience)

#### `<qhorus-workbench>`
Complete chat experience assembled from composites.

**Layout (using pages primitives):**
```
split("horizontal", [
    qhorus-channel-nav,                    // left panel (~20%)
    split("vertical", [
        qhorus-channel-feed,              // main content
        qhorus-message-input              // bottom input
    ]),
    dockBar([                              // right dock strip
        { id: "members",     panel: qhorus-member-panel },
        { id: "tasks",       panel: qhorus-task-panel },
        { id: "artifacts",   panel: qhorus-artifact-panel },
        { id: "correlation", panel: qhorus-correlation-panel },
    ])
])
```

**Responsibilities:**
- Owns the data connection (WebSocket/SSE adapter)
- Configures and distributes datasets to composites
- Handles `chat:*` events from composites → translates to REST/WebSocket
  calls
- Manages responsive behavior via pages layout primitives (split panels
  collapse on smaller viewports)

**Mounting API:**
```typescript
// Full workbench (claudony, chat-demo)
hostPanel("qhorus-workbench", {
    endpoint: "/ws/chat",        // WebSocket URL
    restBase: "/api",            // REST base URL
    spaceId: "case-123",         // optional — filter to space
    channelId: "work",           // optional — open specific channel
})

// Just the feed (embedded in a case detail page)
hostPanel("qhorus-channel-feed", {
    channelId: "case-123-work",
    mode: "threaded",
})
```

---

## 3. Data Flow

### Inbound (qhorus → UI)

```
Qhorus backend (WebSocket or SSE)
    ↓
DataSource adapter (transport-specific)
    ↓
Pages Datasets:
    channels   → qhorus-channel-nav
    messages   → qhorus-channel-feed
    members    → qhorus-member-panel
    presence   → qhorus-member-panel
    reactions  → qhorus-message (per-message)
    topics     → qhorus-channel-feed (topic grouping)
    commitments → qhorus-task-panel
```

Each composite is a LitElement that receives its dataset via the pages
data pipeline (`DataSetManager` event model from `@casehubio/pages-data`).
The workbench owns the data connection and configures the datasets.
Composites don't know the transport.

### Dataset subscription pattern

Composites use **event-based data subscription** — the same integration
pattern as `@casehubio/pages-viz` components (`PagesElement`), adapted
for LitElement:

1. Composite dispatches `pages-data-request` custom event during
   `connectedCallback`, specifying its required dataset IDs
2. The pages runtime (whether a full workbench or standalone `hostPanel`)
   catches the event and resolves datasets via the data pipeline
3. Runtime feeds data back through reactive Lit `@property()` setters on
   the composite (e.g., `messages`, `reactions`, `commitments`)

This preserves two mounting modes:
- **Workbench-hosted:** the workbench configures the data pipeline,
  subscribes to push sources, and the runtime routes datasets to
  composites automatically
- **Standalone-hosted** (`hostPanel("qhorus-channel-feed", {...})`): the
  host application configures its own data pipeline; composites work
  identically because they only depend on the `pages-data-request` event
  contract, not on the workbench

The key difference from `PagesElement` in pages-viz: PagesElement extends
raw `HTMLElement` and uses a `DataReceiver` interface with imperative
setters. Chat composites extend `LitElement` and use reactive `@property()`
declarations — same data flow direction, different lifecycle model.

### Outbound (UI → qhorus)

Components emit `pages-event` topics — they never call REST directly.

| Event topic | Payload | Handler |
|------------|---------|---------|
| `chat:send-message` | `{channelId, content, topic, inReplyTo?, speechAct?}` | Workbench → REST/WebSocket |
| `chat:react` | `{messageId, emoji}` | Workbench → REST |
| `chat:unreact` | `{messageId, emoji}` | Workbench → REST |
| `chat:create-channel` | `{name, description, spaceId?}` | Workbench → REST |
| `chat:delete-channel` | `{channelId}` | Workbench → REST |
| `chat:select-channel` | `{channelId}` | Internal — updates feed dataset filter |
| `chat:select-topic` | `{channelId, topic}` | Internal — filters feed to topic |
| `chat:resolve-topic` | `{channelId, topic}` | Workbench → REST |
| `chat:message-selected` | `{message}` | Internal — updates panels |

### Chat-demo backend adapter

The existing chat-demo backend speaks WebSocket with snapshot/append/replace/
remove ops. An adapter maps this to the pages dataset shape:

| Chat-demo dataset | Pages dataset | Mapping |
|-------------------|---------------|---------|
| channels (id, name, topic) | channels | Direct — add defaults for missing fields |
| messages (channelId, messageId, parentId, senderId, text, timestamp) | messages | parentId → inReplyTo; passthrough messageType if present, default EVENT; passthrough actorType if present, default HUMAN; passthrough topic if present, default "General" |
| reactions (messageId, emoji) | reactions | Direct |
| members (membershipId, channelId, memberId, displayName) | members | Add role=PARTICIPANT |
| presence (memberId, status) | presence | Map ONLINE/AWAY/OFFLINE |

The components render the same way regardless of whether the backend is
qhorus-native or chat-demo. Richer fields (speech acts, topics, artefactRefs)
simply light up when the data includes them.

### Event mechanism

All `chat:*` topics are `pages-event` DOM custom events (`bubbles: true`,
`composed: true`), consistent with the existing pages inter-panel
communication convention. Components dispatch via `emitPagesEvent()` from
`@casehubio/pages-component`.

**Migration from existing chat-demo events:**

| Current (chat-demo) | New (qhorus) | Change |
|---------------------|--------------|--------|
| `ws-data` (WebSocket snapshot/append/remove) | Adapter-managed datasets | Adapter consumes raw WebSocket, applies mutations to `DataSetManager`. Components no longer listen for `ws-data`. |
| `channel-selected` | `chat:select-channel` | Same mechanism (`pages-event`), renamed to `chat:*` namespace. |
| Direct REST via `authenticatedFetch` | `chat:create-channel` etc. → workbench handler | Components emit events; workbench translates to REST. No direct REST in components. |

The `chat:*` namespace is specific to qhorus chat components. It is not a
new platform convention — other pages applications use their own topic
namespaces.

### Capability-driven UI

The workbench adapter exposes capability flags derived from the backend's
supported features. Composites check these flags to show, hide, or disable
UI elements:

| Capability | UI element | When unsupported |
|-----------|-----------|-----------------|
| Messaging | Entire workbench | **Required** — workbench will not mount without it |
| Discovery | Channel nav population | Manual channel ID entry only |
| Members | Member panel list | Member panel hidden |
| Reactions | Reaction bar, add reaction button | Hidden |
| Presence | Presence dots in member panel | Show all as "unknown" status |
| ChannelManagement | Create/delete channel buttons | Hidden |
| MemberManagement | Add/remove member controls | Hidden |
| MessageHistory | Scroll-back loading | Disabled with tooltip |
| Threading | Thread view mode toggle | Hidden; flat mode only |

The adapter declares capabilities at construction time. For ChatPlatform
SPI backends, these map directly to `ChatPlatform.supports()`. For the
chat-demo backend, the adapter hardcodes capabilities based on what its
REST/WebSocket API implements.

### Connection lifecycle

The workbench adapter uses the pages `PushSource` infrastructure
(`@casehubio/pages-data`) for connection management, which provides:

- **Auto-reconnect** with exponential backoff (built into `PushSource`)
- **Error classification** — transient errors (corrupt message, network
  glitch) log and reconnect; permanent errors (auth expired, server
  rejected) propagate to components via `target.error`
- **Pool management** — one connection per base URL, shared across datasets

**UI presentation:**

| Connection state | UI indicator |
|-----------------|-------------|
| Connected | No indicator (normal state) |
| Reconnecting | Subtle banner: "Reconnecting..." with spinner |
| Disconnected (permanent) | Persistent banner: "Connection lost" with retry button |

**Message queue during disconnect:** Messages composed while disconnected
are held in a local queue. On reconnect, queued messages are sent in order.
If reconnection fails permanently, the queue is surfaced to the user with
option to retry or discard.

**State sync on reconnect:** The adapter requests a full dataset snapshot
on reconnection (the `PushSource` protocol supports this via the `snapshot`
op). Components re-render from the fresh snapshot.

---

## 4. UX Layers

### Layer 1: Inline annotations (always visible, lightweight)

Every message in the stream shows:
- Speech act badge (small pill, color-coded by category)
- Actor icon (human/agent/system)
- Sender name + relative timestamp
- Content (markdown)
- Reaction pills (if any)
- Artefact chips (if any artefactRefs)

This is information-dense but visually quiet. Users who don't care about
speech acts can ignore the badges and read the stream as plain text.

In simple chat mode (no qhorus enrichments), speech act badges and actor
icons are hidden — messages render like a standard chat app.

### Layer 2: Progressive disclosure (on interaction)

Click a message to expand:
- Full correlation chain context
- Artefact details with preview
- Commitment state and deadline
- Topic and channel metadata

This is the "drill down" — visible when you care about a specific message.

### Layer 3: Dockable contextual panels (persistent alternative views)

The dockBar provides persistent panels that update based on selected
message or channel context:
- **Members** — who's in this channel, their presence
- **Tasks** — all obligations, their states, click to navigate
- **Artifacts** — referenced documents, code, cases
- **Correlation** — full chain flow diagram

Users customize which panels are visible. Desktop shows multiple. Smaller
viewports collapse to the feed with panels accessible via dock strip toggle.

---

## 5. Port Strategy

### Principle: build alongside, swap atomically, delete old code whole

No period where old and new code coexist in a broken state. No feature
flags. No conditional rendering.

### Directory structure during port

```
chat-demo/src/main/webui/
├── src/
│   ├── index.ts                    ← OLD entry point (untouched until swap)
│   ├── index.html                  ← OLD html
│   ├── auth.ts                     ← KEEP (shared)
│   ├── auth.test.ts                ← KEEP
│   ├── responsive.ts               ← OLD (delete at swap)
│   ├── responsive.test.ts          ← OLD (delete at swap)
│   ├── layout-fit.test.ts          ← OLD (delete at swap)
│   ├── test-helpers.ts             ← OLD (delete at swap)
│   ├── panels/
│   │   ├── channel-sidebar.ts      ← OLD (delete at swap)
│   │   ├── message-list.ts         ← OLD (delete at swap)
│   │   ├── message-input.ts        ← OLD (delete at swap)
│   │   └── member-list.ts          ← OLD (delete at swap)
│   └── qhorus/                     ← NEW (all new code here)
│       ├── primitives/
│       │   ├── qhorus-message.ts
│       │   ├── qhorus-message.test.ts
│       │   ├── qhorus-message-input.ts
│       │   ├── qhorus-message-input.test.ts
│       │   ├── qhorus-thread.ts
│       │   ├── qhorus-thread.test.ts
│       │   ├── qhorus-reaction-bar.ts
│       │   └── qhorus-reaction-bar.test.ts
│       ├── composites/
│       │   ├── qhorus-channel-feed.ts
│       │   ├── qhorus-channel-feed.test.ts
│       │   ├── qhorus-channel-nav.ts
│       │   ├── qhorus-channel-nav.test.ts
│       │   ├── qhorus-member-panel.ts
│       │   ├── qhorus-member-panel.test.ts
│       │   ├── qhorus-artifact-panel.ts
│       │   ├── qhorus-task-panel.ts
│       │   └── qhorus-correlation-panel.ts
│       ├── workbench/
│       │   ├── qhorus-workbench.ts
│       │   ├── qhorus-workbench.test.ts
│       │   └── chat-demo-adapter.ts  ← maps chat-demo backend to datasets
│       ├── types.ts                  ← QhorusMessage, Reaction, etc.
│       └── index.ts                  ← NEW entry point
```

### Build steps

| Step | What | Tests | Old code touched? |
|------|------|-------|:-:|
| 1. Scaffold | Package structure, Lit + pages + blocks-ui-core deps, vitest config | Config validates | No |
| 2. Types | `types.ts` — QhorusMessage, QhorusChannel, Reaction, CommitmentState, ArtefactRef | Type tests | No |
| 3. Primitives | qhorus-message, reaction-bar, thread, message-input | Props → rendered output | No |
| 4. Composites | channel-feed, channel-nav, member-panel | Mock datasets → rendered output | No |
| 5. Workbench + adapter | qhorus-workbench, chat-demo-adapter | Live backend integration | No |
| 6. Swap | New index.ts replaces old. Delete old files. | Full regression | Yes — delete |
| 7. Panels | artifact-panel, task-panel, correlation-panel | Mock data → rendered output | No |

### Build tooling change

Current: esbuild (ESM bundle to dist/app.js). New: adopt Vite for dev
server + HMR, keep esbuild for production bundle (Vite uses esbuild
internally). Add Lit as a dependency. The build config is part of the
scaffold step. Package structure follows blocks-ui conventions
(component per directory, co-located tests) so the code is extractable
to blocks-ui if the composition argument materialises later.

---

## 6. Phased Epics

### Phase 1: Foundation
**Depends on:** Nothing (uses existing qhorus fields only).
**Blocked by:** Message rendering format decision (§9 Q2) — must resolve
before build step 3 (primitives).

Port steps 1–6. Primitives, composites, workbench with pages layout.
Speech act badges, actor indicators, correlation chain grouping, flat and
threaded view modes. Works with chat-demo backend via adapter AND qhorus
channels as-is. Topic view mode deferred to Phase 4 (requires `topic`
field from qhorus#328).

Replaces the current webui entirely. Old code deleted.

### Phase 2: Workbench & Panels
**Depends on:** Phase 1

Port step 7. Dockable contextual panels: artifact viewer, task/commitment
panel, correlation flow panel, member panel with presence. Progressive
disclosure on messages.

### Phase 3: Rich Anchoring & Reactions
**Depends on:** Phase 2, qhorus#328 (ArtefactRef, Reactions)

Rich artefact references with selection scope. Document side viewer with
anchor navigation. Emoji reactions with palette. Drafthouse integration
(clicking debate references opens in artifact panel).

### Phase 4: Channel Hierarchy & Topics
**Depends on:** Phase 2, qhorus#328 (Space, Topic, Membership, Presence)

Tree-based channel navigation with space grouping. Topics as named
persistent sub-conversations. Presence indicators. Channel membership
model distinct from ACL. Normative triple grouped view.

### Phase 5: Claudony Integration
**Depends on:** Phase 2 minimum, Phase 4 ideal

Embed the workbench in claudony. Wire to live qhorus channels via SSE.
Normative triple view (work/observe/oversight as grouped case node).
Human participation via oversight channel.

---

## 7. Accessibility

All components use Lit accessibility mixins from
`@casehubio/blocks-ui-core` (`packages/blocks-ui-core/src/mixins/`).
These are Lit 3.3 mixins with tests. The mixins are planned to migrate
to `@casehubio/pages-primitives` (per pages ARC42STORIES §5/§10); when
that migration happens, import paths change but APIs do not.

- **KeyboardShortcutMixin** — keyboard navigation throughout (arrow keys
  in channel nav, Escape to close panels, Enter to send)
- **LiveRegionMixin** — screen reader announcements for new messages,
  channel changes, topic updates
- **RovingTabindexMixin** — keyboard-navigable lists (channel list, member
  list, topic bar)
- **FocusTrapMixin** — modal dialogs (create channel, confirm delete)
- **`prefers-reduced-motion`** — all animations disabled when preference set
- **`aria-expanded`**, **`inert`** — on collapsible threads and dockable panels

---

## 8. Styling

All components use `--pages-*` design tokens exclusively:

- Colors: `--pages-accent-*`, `--pages-neutral-*`, `--pages-success-*`,
  `--pages-danger-*`, `--pages-warning-*`, `--pages-info-*` (12-step OKLCH)
- Spacing: `--pages-space-*` scale
- Typography: `--pages-font-*`, `--pages-font-size-*`, `--pages-line-height-*`
- Motion: `--pages-duration-*`, `--pages-ease-*`
- Elevation: `--pages-shadow-*`, `--pages-radius-*`

No hardcoded colors, fonts, or spacing values. Dark mode works
automatically via the OKLCH token system. Compact density supported via
`pages-density-compact` class.

Speech act badge colors mapped to semantic token groups:
- Information exchange (QUERY, RESPONSE, STATUS) → `--pages-info-*`
- Obligation lifecycle (COMMAND) → `--pages-accent-*`
- Terminal success (DONE) → `--pages-success-*`
- Terminal failure (FAILURE) → `--pages-danger-*`
- Terminal refusal (DECLINE) → `--pages-warning-*`
- Obligation transfer (HANDOFF) → `--pages-info-*` (distinct from
  DECLINE — a transfer is not a refusal)
- Telemetry (EVENT) → `--pages-neutral-*`

Commitment state badge colors:
- Active: OPEN → `--pages-accent-*`, ACKNOWLEDGED → `--pages-info-*`
- Success: FULFILLED → `--pages-success-*`
- Failure: FAILED → `--pages-danger-*`
- Refusal: DECLINED → `--pages-neutral-*`
- Transfer: DELEGATED → `--pages-info-*` (matches HANDOFF speech act)
- Timeout: EXPIRED → `--pages-warning-*`

---

## 9. Open Questions

1. **Emoji palette component:** Build custom or adopt an existing Lit-based
   emoji picker? Deferred to Phase 3.
2. **Message rendering:** Markdown vs plain text vs rich content? Current
   chat-demo is plain text. Qhorus messages may contain structured content.
   Blocks Phase 1 implementation — must decide before primitives.

### Settled Decisions

- **Build tooling:** Vite for dev server + HMR, esbuild for production
  bundle (Vite uses esbuild internally). Aligns with blocks-ui convention.
- **Lit version:** Lit 3.3, matching `@casehubio/blocks-ui-core` (per
  pages ARC42STORIES §10).
- **WebSocket vs SSE:** The adapter pattern supports both. Chat-demo uses
  WebSocket; claudony uses SSE. No single primary — each backend provides
  its adapter.
