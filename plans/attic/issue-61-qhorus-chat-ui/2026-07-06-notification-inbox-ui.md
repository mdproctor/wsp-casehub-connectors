# Notification Inbox UI Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** casehubio/platform#146 — notification center frontend
**Issue group:** platform#147 (epic), pages#131 (SSEManager named events, done), blocks-ui#22 (data-table, finishing)

**Goal:** Build 5 composable Lit web components in blocks-ui for the CaseHub notification system — notification bell, notification inbox, subscription list, subscription editor, and contextual subscribe button.

**Architecture:** Components live in `blocks-ui/components/notification-inbox/` as a new workspace package. They reuse pages-primitives (`PagesFilterChips`, `PagesScopeSelector`, a11y mixins), pages-data (`SSEManager` with named event support), blocks-ui-core (`BlocksConfirmDialog`, design tokens), and `pages-data-table` for row display. Only notification-specific orchestration and domain logic is new.

**Tech Stack:** Lit 3.x, TypeScript 5.6+, vitest, `@casehubio/pages-data` (SSEManager), `@casehubio/blocks-ui-core`, `@casehubio/pages-primitives` (when available via workspace)

## Global Constraints

- **Web protocols:** All CustomEvents crossing shadow DOM: `{ bubbles: true, composed: true }` (protocol: `custom-event-shadow-dom`). All reactive Set/Map/Array mutations: replace reference, never mutate in place (protocol: `lit-immutable-collections`).
- **SSEManager:** Import from `@casehubio/pages-data`, not `@casehubio/blocks-ui-core`. Use `eventNames` parameter for named SSE events.
- **Design tokens:** Use `--blocks-*` / `--pages-*` CSS variables. Never hardcode colours, spacing, or typography values.
- **Events:** Follow `pages-event` topic convention for inter-component communication. Internal events use `composed: false`.
- **Endpoint guard:** Check `this.endpoint != null` not `if (this.endpoint)` — empty string is a valid same-origin base (garden: GE-20260705-1cda0b).
- **Response validation:** Validate REST response shape before casting — a single TypeError in the render pipeline locks the UI (garden: GE-20260705-557ee5).
- **No backwards compatibility shims.** Breaking changes cost nothing. Fix the design.
- **TDD:** Write failing test → verify fail → implement → verify pass → commit. Every task.
- **Code review:** `requesting-code-review` before every commit.
- **Repo:** All work in `~/claude/casehub/blocks-ui` on a feature branch.

---

## File Structure

All paths relative to `blocks-ui/components/notification-inbox/`.

### New files (create)

| File | Responsibility |
|------|---------------|
| `package.json` | Package manifest with workspace deps |
| `tsconfig.json` | TypeScript config with project references |
| `tsconfig.build.json` | Build config (excludes tests) |
| `vitest.config.ts` | Test config matching blocks-ui convention |
| `src/index.ts` | Public exports |
| `src/types.ts` | Notification, Subscription, Constraint, NotificationSeverity, NotificationStatus, MuteRule, Snooze, ChannelPreference, EventTypeDescriptor types |
| `src/api.ts` | REST client — typed fetch wrappers for all notification/subscription/preference/mute/snooze endpoints |
| `src/events.ts` | Event topic constants + `emitNotificationEvent` helper |
| `src/notification-bell.ts` | `<notification-bell>` — toolbar bell with badge + dropdown |
| `src/notification-bell.test.ts` | Tests |
| `src/notification-inbox.ts` | `<notification-inbox>` — container orchestrator |
| `src/notification-inbox.test.ts` | Tests |
| `src/subscription-list.ts` | `<subscription-list>` — personal subscription CRUD |
| `src/subscription-list.test.ts` | Tests |
| `src/subscription-editor.ts` | `<subscription-editor>` — constraint builder + event type picker |
| `src/subscription-editor.test.ts` | Tests |
| `src/subscribe-button.ts` | `<subscribe-button>` — contextual subscribe |
| `src/subscribe-button.test.ts` | Tests |

### Modified files (in blocks-ui root)

| File | Change |
|------|--------|
| `tsconfig.json` | Add `{ "path": "components/notification-inbox" }` to references |

---

## Task 1: Package Scaffold + Types + API Client

**Files:**
- Create: `components/notification-inbox/package.json`
- Create: `components/notification-inbox/tsconfig.json`
- Create: `components/notification-inbox/tsconfig.build.json`
- Create: `components/notification-inbox/vitest.config.ts`
- Create: `components/notification-inbox/src/types.ts`
- Create: `components/notification-inbox/src/api.ts`
- Create: `components/notification-inbox/src/api.test.ts`
- Create: `components/notification-inbox/src/events.ts`
- Create: `components/notification-inbox/src/index.ts`
- Modify: `tsconfig.json` (root)

**Interfaces:**
- Consumes: Platform REST API types (Notification, Subscription, etc. from platform-api Java records — translate to TypeScript)
- Produces: `NotificationApi` class (typed REST client), all TypeScript domain types, event topic constants. Every subsequent task imports from here.

### Steps

- [ ] **Step 1: Create package.json**

```json
{
  "name": "@casehubio/blocks-ui-notification-inbox",
  "version": "0.0.1",
  "type": "module",
  "main": "dist/index.js",
  "types": "dist/index.d.ts",
  "files": ["dist"],
  "scripts": {
    "build": "tsc --project tsconfig.build.json",
    "test": "vitest run"
  },
  "dependencies": {
    "@casehubio/blocks-ui-core": "workspace:*",
    "@casehubio/pages-data": "^0.2.1",
    "lit": "^3.0.0"
  },
  "devDependencies": {
    "@open-wc/testing": "^4.0.0",
    "jsdom": "^25.0.0",
    "typescript": "^5.6.0",
    "vitest": "^3.0.0"
  }
}
```

- [ ] **Step 2: Create tsconfig.json**

```json
{
  "extends": "../../tsconfig.base.json",
  "compilerOptions": {
    "composite": true,
    "outDir": "dist",
    "rootDir": "src",
    "experimentalDecorators": true,
    "useDefineForClassFields": false
  },
  "include": ["src/**/*"],
  "references": [
    { "path": "../../packages/blocks-ui-core" },
    { "path": "../data-table" }
  ]
}
```

- [ ] **Step 3: Create tsconfig.build.json**

```json
{
  "extends": "./tsconfig.json",
  "compilerOptions": {
    "declaration": true,
    "declarationMap": true,
    "sourceMap": true
  },
  "exclude": ["src/**/*.test.ts"]
}
```

- [ ] **Step 4: Create vitest.config.ts**

```typescript
import { defineConfig } from 'vitest/config';

export default defineConfig({
  test: {
    environment: 'jsdom',
    include: ['src/**/*.test.ts'],
  },
});
```

- [ ] **Step 5: Create src/types.ts**

Translate platform-api Java records to TypeScript. Cross-reference with the actual Java types via IntelliJ MCP: `ide_find_class` for `Notification`, `Subscription`, `Constraint`, `NotificationTarget`, `MuteRule`, `Snooze`, `NotificationPreferences`, `ChannelPreference`, `QuietHours`, `EventTypeDescriptor`, `EventFieldDescriptor`, `DeliveryChannelDescriptor` in `/Users/mdproctor/claude/casehub/platform`. Read each record's fields and translate faithfully. Key types:

```typescript
export type NotificationStatus = 'UNREAD' | 'READ' | 'DISMISSED';
export type NotificationSeverity = 'INFO' | 'WARNING' | 'URGENT';
export type ConstraintOp = 'EQ' | 'NEQ' | 'GT' | 'LT' | 'GTE' | 'LTE' | 'IN' | 'STARTS_WITH' | 'CONTAINS';
export type TargetType = 'USER' | 'GROUP' | 'EVENT_FIELD';
export type MuteScope = 'ENTITY' | 'CATEGORY';

export interface Notification {
  readonly id: string;
  readonly userId: string;
  readonly tenancyId: string;
  readonly title: string;
  readonly body: string | null;
  readonly category: string;
  readonly severity: NotificationSeverity;
  readonly actionUrl: string | null;
  readonly source: NotificationSource;
  readonly status: NotificationStatus;
  readonly createdAt: string;
  readonly readAt: string | null;
  readonly dismissedAt: string | null;
}

export interface NotificationSource {
  readonly eventId: string;
  readonly entityType: string;
  readonly entityId: string;
  readonly actorId: string;
}

export interface NotificationPage {
  readonly notifications: readonly Notification[];
  readonly nextCursor: string | null;
}

export interface Constraint {
  readonly field: string;
  readonly op: ConstraintOp;
  readonly value: string;
}

export interface NotificationTarget {
  readonly type: TargetType;
  readonly id: string;
}

export interface NotificationTemplate {
  readonly titlePattern: string;
  readonly bodyPattern: string | null;
  readonly severity: NotificationSeverity;
  readonly category: string;
  readonly actionUrlPattern: string | null;
  readonly entityType: string;
  readonly entityIdField: string;
  readonly actorIdField: string;
}

export interface Subscription {
  readonly id: string;
  readonly ownerId: string;
  readonly tenancyId: string;
  readonly name: string;
  readonly eventType: string;
  readonly constraints: readonly Constraint[];
  readonly targets: readonly NotificationTarget[];
  readonly excludeActor: boolean;
  readonly template: NotificationTemplate;
  readonly enabled: boolean;
  readonly createdAt: string;
  readonly updatedAt: string;
}

export interface SubscriptionPage {
  readonly subscriptions: readonly Subscription[];
  readonly nextCursor: string | null;
}

export interface MuteRule {
  readonly id: string;
  readonly userId: string;
  readonly tenancyId: string;
  readonly scope: MuteScope;
  readonly scopeId: string;
  readonly entityType: string;
  readonly createdAt: string;
  readonly expiresAt: string | null;
}

export interface Snooze {
  readonly userId: string;
  readonly tenancyId: string;
  readonly until: string;
  readonly createdAt: string;
}

export interface ChannelPreference {
  readonly enabled: boolean;
  readonly minSeverity: NotificationSeverity;
}

export interface QuietHours {
  readonly start: string;
  readonly end: string;
  readonly timezone: string;
}

export interface NotificationPreferences {
  readonly userId: string;
  readonly tenancyId: string;
  readonly channelDefaults: Record<string, ChannelPreference>;
  readonly quietHours: QuietHours | null;
  readonly updatedAt: string;
}

export interface EventTypeDescriptor {
  readonly eventType: string;
  readonly displayName: string;
  readonly description: string;
  readonly fields: readonly EventFieldDescriptor[];
}

export interface EventFieldDescriptor {
  readonly name: string;
  readonly displayName: string;
  readonly type: string;
}

export interface DeliveryChannelDescriptor {
  readonly channelId: string;
  readonly displayName: string;
  readonly external: boolean;
}

export interface SSENotificationEvent {
  readonly type: 'notification' | 'notification-updated' | 'unread-count';
  readonly data: unknown;
}
```

- [ ] **Step 6: Create src/events.ts**

```typescript
export const NotificationEventTopics = {
  SELECTED: 'notification.selected',
  DISMISSED: 'notification.dismissed',
  MUTED: 'notification.muted',
  SUBSCRIPTION_CREATED: 'subscription.created',
  SUBSCRIPTION_DELETED: 'subscription.deleted',
} as const;

export function emitNotificationEvent<T>(
  target: EventTarget,
  topic: string,
  payload: T,
): void {
  target.dispatchEvent(
    new CustomEvent('pages-event', {
      bubbles: true,
      composed: true,
      detail: { topic, payload },
    }),
  );
}
```

- [ ] **Step 7: Create src/api.ts with tests**

The API client wraps all REST endpoints with typed fetch. Use `fetch` directly (not `authenticatedFetch` from chat-demo — that's a connectors pattern; blocks-ui components accept auth via SSEManager config or a fetch wrapper prop).

Write `src/api.test.ts` first: test that each method calls the correct URL/method/body and parses the response. Use `vi.fn()` to mock `fetch`.

Key API methods:

```typescript
export class NotificationApi {
  constructor(private readonly baseUrl: string, private readonly fetchFn: typeof fetch = fetch) {}

  async listNotifications(params: { status?: string; category?: string; cursor?: string; limit?: number }): Promise<NotificationPage>
  async unreadCount(): Promise<number>
  async markRead(id: string): Promise<Notification>
  async dismiss(id: string): Promise<Notification>
  async markAllRead(): Promise<number>
  async listSubscriptions(params?: { enabled?: boolean; cursor?: string; limit?: number }): Promise<SubscriptionPage>
  async createSubscription(input: SubscriptionInput): Promise<Subscription>
  async updateSubscription(id: string, update: SubscriptionUpdate): Promise<Subscription>
  async deleteSubscription(id: string): Promise<void>
  async enableSubscription(id: string): Promise<Subscription>
  async disableSubscription(id: string): Promise<Subscription>
  async getEventTypes(): Promise<readonly EventTypeDescriptor[]>
  async getPreferences(): Promise<NotificationPreferences | null>
  async updatePreferences(update: NotificationPreferenceUpdate): Promise<NotificationPreferences>
  async addMuteRule(input: MuteRuleInput): Promise<MuteRule>
  async listMuteRules(): Promise<readonly MuteRule[]>
  async removeMuteRule(id: string): Promise<void>
  async activateSnooze(until: string): Promise<Snooze>
  async getSnooze(): Promise<Snooze | null>
  async cancelSnooze(): Promise<void>
  async getChannels(): Promise<readonly DeliveryChannelDescriptor[]>
}
```

Each method: construct URL, call `fetchFn`, validate response status, parse JSON, validate shape (check for expected fields before returning). On non-2xx: throw a typed `ApiError` with status and message.

- [ ] **Step 8: Create src/index.ts**

```typescript
export type * from './types.js';
export { NotificationApi } from './api.js';
export { NotificationEventTopics, emitNotificationEvent } from './events.js';
// Component exports added in subsequent tasks
```

- [ ] **Step 9: Register in root tsconfig.json**

Add `{ "path": "components/notification-inbox" }` to the root `tsconfig.json` references array.

- [ ] **Step 10: Run tests, verify pass**

```bash
cd components/notification-inbox && npx vitest run
```

- [ ] **Step 11: Commit**

```bash
git add components/notification-inbox/ tsconfig.json
git commit -m "feat(notification-inbox): package scaffold, types, API client, events"
```

---

## Task 2: notification-bell Component

**Files:**
- Create: `src/notification-bell.ts`
- Create: `src/notification-bell.test.ts`
- Modify: `src/index.ts` (add export)

**Interfaces:**
- Consumes: `SSEManager` from `@casehubio/pages-data`, `NotificationApi` from Task 1, `WorkIdentity` from `@casehubio/blocks-ui-core`
- Produces: `<notification-bell>` custom element. Properties: `endpoint`, `identity`, `open`. Events: `pages-event` with topic `notification.selected` (bubbles from inner inbox).

### Steps

- [ ] **Step 1: Write failing tests**

Test file `src/notification-bell.test.ts`. Tests:

1. `renders bell icon with no badge when unreadCount is 0`
2. `shows badge with count when unreadCount > 0`
3. `shows 99+ when unreadCount exceeds 99`
4. `toggles dropdown on click`
5. `closes dropdown on Escape key`
6. `subscribes to SSE on connectedCallback with named events`
7. `updates unreadCount from SSE unread-count event`
8. `increments count optimistically on SSE notification event`
9. `aria-label includes count when unread`
10. `aria-expanded reflects open state`

Mock `SSEManager` — create a `MockSSEManager` that captures the subscribe call and exposes `emit(type, data)` to simulate SSE events. Check blocks-ui-core for existing mock patterns (the data-table tests have `MockSSESource`).

- [ ] **Step 2: Run tests to verify they fail**

```bash
cd components/notification-inbox && npx vitest run src/notification-bell.test.ts
```

Expected: FAIL — `notification-bell` not defined.

- [ ] **Step 3: Implement notification-bell.ts**

Lit component extending `FocusTrapMixin(KeyboardShortcutMixin(LitElement))` from `@casehubio/pages-primitives` (if available via workspace) or `@casehubio/blocks-ui-core`.

Key implementation points:
- `connectedCallback`: subscribe SSEManager with `eventNames: ['notification', 'notification-updated', 'unread-count']`
- `disconnectedCallback`: unsubscribe SSEManager
- SSE handler: switch on `event.type` — `unread-count` sets badge, `notification` increments optimistically
- Render: bell SVG + conditional badge span + conditional dropdown with `<notification-inbox>`
- FocusTrap on dropdown when open
- Escape closes dropdown, returns focus to bell button
- Badge: `--pages-danger-9` background, `aria-hidden="true"` (count in button's `aria-label`)

- [ ] **Step 4: Run tests to verify they pass**

- [ ] **Step 5: Add export to index.ts**

```typescript
export { NotificationBell } from './notification-bell.js';
```

- [ ] **Step 6: Commit**

```bash
git commit -m "feat(notification-inbox): notification-bell — SSE badge + dropdown"
```

---

## Task 3: notification-inbox Component

**Files:**
- Create: `src/notification-inbox.ts`
- Create: `src/notification-inbox.test.ts`
- Modify: `src/index.ts` (add export)

**Interfaces:**
- Consumes: `NotificationApi` from Task 1, `PagesDataTable` + `ColumnDef` from `@casehubio/blocks-ui-data-table`, `PagesFilterChips` from `@casehubio/pages-primitives` (or inline implementation if not yet available as workspace dep), `WorkIdentity` from `@casehubio/blocks-ui-core`
- Produces: `<notification-inbox>` custom element. Properties: `endpoint`, `data`, `identity`. Events: `pages-event` with `notification.selected`, `notification.dismissed`, `notification.muted`.

### Steps

- [ ] **Step 1: Write failing tests**

Tests:

1. `renders Inbox tab as default active scope`
2. `switches to Archive tab and re-fetches with DISMISSED status`
3. `renders filter chips with category counts from data`
4. `filters items client-side when category chip toggled`
5. `renders notification rows via pages-data-table with custom cell renderer`
6. `shows severity left border via getRowClass`
7. `shows unread dot for UNREAD notifications`
8. `emits notification.selected on row-activate`
9. `marks notification read via API on row click (optimistic)`
10. `rolls back optimistic update on API failure`
11. `dismisses notification via API (optimistic, moves to archive)`
12. `shows batch action bar when 2+ items selected`
13. `batch mark-read processes selected items in parallel`
14. `batch dismiss shows BlocksConfirmDialog before execution`
15. `loads next page via cursor on load-more event`
16. `prepends SSE notification events, deduplicates by id`
17. `updates local item on SSE notification-updated event`
18. `announces new notifications via LiveRegionMixin`

Provide mock data: 5 notifications with different severities, categories, statuses. Mock `NotificationApi` methods with `vi.fn()`.

- [ ] **Step 2: Run tests to verify they fail**

- [ ] **Step 3: Implement notification-inbox.ts**

Lit component extending `KeyboardShortcutMixin(LiveRegionMixin(LitElement))`.

Key implementation points:
- `NotificationApi` instance created from `endpoint` prop
- Column definitions with `ColumnDef<Notification>` — title (custom render: title + body), category, status (unread dot), createdAt (relative time). Use `render` callback for custom cells.
- `getRowClass: (n) => \`severity-${n.severity.toLowerCase()}\`` for left border
- `getRowKey: (n) => n.id` for selection
- Filter chips: combine category chips (dynamic from data), severity chips (static), read-state chip (inbox tab only). Prefix ids with `cat:`, `sev:`, `read:` for disambiguation.
- Tab switch via `PagesScopeSelector` (if available) or inline tab buttons
- Scroll mode with `hasMore` based on cursor presence
- SSE: if `endpoint` set and not using static `data`, subscribe via `SSEManager` with named events. On `notification`: prepend, deduplicate by id. On `notification-updated`: find and replace. On `unread-count`: update summary.
- Batch actions: floating bar when `selectedItems.size >= 2`
- Optimistic updates: snapshot before mutate, restore on failure
- Keyboard shortcuts: `d` dismiss, `m` mute (via `KeyboardShortcutMixin`)
- Collection mutations: always create new Set/Map (protocol: `lit-immutable-collections`)

CSS: scoped styles with `--blocks-*` tokens. Row severity borders via `::part()`:
```css
pages-data-table::part(severity-urgent) { border-left: 3px solid var(--pages-danger-9); }
pages-data-table::part(severity-warning) { border-left: 3px solid var(--pages-warning-9); }
pages-data-table::part(severity-info) { border-left: 3px solid var(--pages-accent-9); }
```

- [ ] **Step 4: Run tests to verify they pass**

- [ ] **Step 5: Add export, commit**

```bash
git commit -m "feat(notification-inbox): inbox container — table, filters, SSE, batch actions"
```

---

## Task 4: subscription-list Component

**Files:**
- Create: `src/subscription-list.ts`
- Create: `src/subscription-list.test.ts`
- Modify: `src/index.ts`

**Interfaces:**
- Consumes: `NotificationApi` from Task 1, `PagesDataTable`, `BlocksConfirmDialog` from `@casehubio/blocks-ui-core`
- Produces: `<subscription-list>` custom element. Properties: `endpoint`, `identity`. Opens `<subscription-editor>` (Task 5) when editing.

### Steps

- [ ] **Step 1: Write failing tests**

1. `fetches and renders subscriptions on connect`
2. `shows subscription name, event type pill, constraint count per row`
3. `toggles subscription enabled state via API`
4. `opens subscription-editor in create mode on New button`
5. `opens subscription-editor in edit mode on Edit button`
6. `shows BlocksConfirmDialog before delete`
7. `deletes subscription after confirmation`
8. `shows system subscriptions as read-only with System badge`
9. `emits subscription.deleted event on delete`

- [ ] **Step 2: Run tests to verify fail**

- [ ] **Step 3: Implement subscription-list.ts**

Lit component. Renders subscriptions via `pages-data-table` with columns: name, event type, constraint count, enabled toggle, actions (edit/delete buttons). The enabled toggle calls `api.enableSubscription` / `api.disableSubscription`. Delete uses `BlocksConfirmDialog`.

When `editing` state is non-null, renders `<subscription-editor>` below the list.

- [ ] **Step 4: Run tests to verify pass**

- [ ] **Step 5: Add export, commit**

```bash
git commit -m "feat(notification-inbox): subscription-list — CRUD with table, toggle, confirm"
```

---

## Task 5: subscription-editor Component

**Files:**
- Create: `src/subscription-editor.ts`
- Create: `src/subscription-editor.test.ts`
- Modify: `src/index.ts`

**Interfaces:**
- Consumes: `NotificationApi` from Task 1 (`getEventTypes`, `createSubscription`, `updateSubscription`), `EventTypeDescriptor` / `EventFieldDescriptor` types
- Produces: `<subscription-editor>` custom element. Properties: `subscription` (null = create), `endpoint`, `identity`. Events: `save`, `cancel`.

### Steps

- [ ] **Step 1: Write failing tests**

1. `renders empty form when subscription is null (create mode)`
2. `populates form fields when subscription provided (edit mode)`
3. `fetches event types from API and populates dropdown`
4. `loads field descriptors when event type selected`
5. `adds constraint row on Add Filter button`
6. `removes constraint row on remove button`
7. `field dropdown populated from EventFieldDescriptor`
8. `op dropdown shows all ConstraintOp values`
9. `adds target row on Add Target button`
10. `defaults target to USER with current identity userId`
11. `EVENT_FIELD target shows field dropdown from event type`
12. `excludeActor checkbox defaults to true`
13. `calls createSubscription on save when creating`
14. `calls updateSubscription on save when editing`
15. `emits save event with subscription on success`
16. `emits cancel event on cancel button`
17. `shows validation error when name is empty`
18. `shows validation error when no event type selected`

- [ ] **Step 2: Run tests to verify fail**

- [ ] **Step 3: Implement subscription-editor.ts**

Lit component. Sections:

1. **Name:** text input bound to local state
2. **Event Type:** `<select>` populated from `api.getEventTypes()`. On change, store selected `EventTypeDescriptor` and its `fields`.
3. **Constraint Builder:** dynamic array of `{field, op, value}` rows. Field dropdown from `EventFieldDescriptor.name`/`displayName`. Op dropdown from `ConstraintOp` enum. Value is free text. Add/remove buttons. Array mutations create new arrays (immutable protocol).
4. **Targets:** dynamic array of `{type, id}` rows. Type dropdown: USER/GROUP/EVENT_FIELD. ID input varies by type — text for USER/GROUP, select from fields for EVENT_FIELD.
5. **Exclude Actor:** checkbox, default true.
6. **Save/Cancel:** validate, then POST or PATCH via API. Emit `save` or `cancel`.

- [ ] **Step 4: Run tests to verify pass**

- [ ] **Step 5: Add export, commit**

```bash
git commit -m "feat(notification-inbox): subscription-editor — event type picker, constraint builder, targets"
```

---

## Task 6: subscribe-button Component

**Files:**
- Create: `src/subscribe-button.ts`
- Create: `src/subscribe-button.test.ts`
- Modify: `src/index.ts`

**Interfaces:**
- Consumes: `NotificationApi` from Task 1
- Produces: `<subscribe-button>` custom element. Properties: `endpoint`, `identity`, `entityType`, `entityId`, `constraints`, `eventType`. Events: `subscription-created`.

### Steps

- [ ] **Step 1: Write failing tests**

1. `renders subscribe icon/button`
2. `shows active state when matching subscription exists`
3. `opens popover on click with pre-filled constraints (entity mode)`
4. `opens popover on click with passed constraints (filter mode)`
5. `creates subscription via API on confirm`
6. `emits subscription-created event on success`
7. `checks for existing matching subscription on mount`

- [ ] **Step 2: Run tests to verify fail**

- [ ] **Step 3: Implement subscribe-button.ts**

Lit component. Two modes:

**Entity mode** (`entityType` + `entityId`): pre-fills constraint `{field: entityType+"Id", op: "EQ", value: entityId}`. Compact popover with name input + confirm.

**Filter mode** (`constraints` + `eventType`): pre-fills from passed props. Popover with name input + editable constraints + confirm.

On mount: check `api.listSubscriptions()` for matching `eventType` — if found, show filled/active state.

On confirm: `api.createSubscription()` with assembled input. Emit `subscription-created`.

- [ ] **Step 4: Run tests to verify pass**

- [ ] **Step 5: Add export, commit**

```bash
git commit -m "feat(notification-inbox): subscribe-button — contextual entity/filter subscribe"
```

---

## Task 7: Integration Test + Final Polish

**Files:**
- Modify: `src/index.ts` (verify all exports)
- Create: `src/integration.test.ts` (optional — tests bell → inbox → subscription flow)

**Interfaces:**
- Consumes: all components from Tasks 2-6
- Produces: verified, buildable package

### Steps

- [ ] **Step 1: Run full test suite**

```bash
cd components/notification-inbox && npx vitest run
```

All tests must pass.

- [ ] **Step 2: Type check**

```bash
cd components/notification-inbox && npx tsc --noEmit
```

No type errors.

- [ ] **Step 3: Build**

```bash
cd components/notification-inbox && npx tsc --project tsconfig.build.json
```

Verify `dist/` contains `.js` + `.d.ts` for all source files.

- [ ] **Step 4: Verify index.ts exports all public types and components**

```typescript
// src/index.ts should export:
export type * from './types.js';
export { NotificationApi } from './api.js';
export { NotificationEventTopics, emitNotificationEvent } from './events.js';
export { NotificationBell } from './notification-bell.js';
export { NotificationInbox } from './notification-inbox.js';
export { SubscriptionList } from './subscription-list.js';
export { SubscriptionEditor } from './subscription-editor.js';
export { SubscribeButton } from './subscribe-button.js';
```

- [ ] **Step 5: Code review**

Run `requesting-code-review`. Any finding Minor or above → GitHub issue.

- [ ] **Step 6: Final commit**

```bash
git commit -m "feat(notification-inbox): integration verification + exports"
```

---

## Deferred Issues to File

Before closing the branch, file these as GitHub issues:

| Issue | Repo | Description |
|-------|------|-------------|
| `pages-summary-bar` primitive | casehub-pages | Generic clickable badge-count bar for pages-primitives. API: `SummaryItem{id, label, count, color, active}`. Replaces blocks-ui `inbox-summary-bar` and notification summary. |
| Snooze/mute popover | blocks-ui | Extract snooze duration picker and mute scope selector as reusable primitives if pattern emerges across components. |
| Notification preferences panel | blocks-ui | Channel preference management UI — per-channel toggles + quiet hours. Deferred from this spec. |
| Digest schedule UI | platform#146 | Platform#146 acceptance criteria include digest schedule config. Cannot fully close #146 until this ships. Depends on platform#144. |
| Chat-demo migration | connectors | Port chat-demo to same primitives (PagesFilterChips, PagesScopeSelector, pages-data-table). |
