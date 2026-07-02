# Chat-Demo Responsive Layout — Design Spec

**Issue:** casehubio/connectors#54
**Date:** 2026-07-02
**Approach:** CSS media queries + lightweight JS coordinator (Approach A)

## Problem

The chat-demo UI renders a fixed three-column layout (rooms | chat | members) that is unusable on phones and tablets. The layout must adapt to three device classes while preserving all existing functionality.

## Breakpoints

| Mode | Viewport | Behaviour |
|------|----------|-----------|
| Phone | `< 768px` | Full-screen chat. Sidebars become slide-in drawers triggered by header buttons. |
| Tablet | `768px – 1023px` | Single sidebar (left) with tab switcher toggling between channels and members. |
| Desktop | `>= 1024px` | Three-column layout. No change from current behaviour. |

Detection: `window.matchMedia` with two queries. A CSS class (`phone`, `tablet`, or `desktop`) is set on `#app`.

## Layout Tree Changes

Three `withId()` calls added to `index.ts` for stable CSS targeting:

```typescript
const chatApp = columns([0, 1],
  [withId("dock", dockBar("vertical", [...]))],
  [withId("main-split", split("horizontal", [
    withId("channel-panel", hostPanel("channels")),
    withId("chat-area", split("vertical", [
      hostPanel("messages"),
      hostPanel("input"),
    ], { ratio: [90, 10] })),
    withId("member-panel", hostPanel("members")),
  ], { ratio: [20, 60, 20] }))],
);
```

New IDs: `dock`, `main-split`, `chat-area`. CSS targets via `[data-component-id="..."]`. No functional change to desktop.

## Phone Layout

The three-column layout collapses to a full-screen chat view with two slide-in drawers.

### CSS transformation

- Outer `columns` grid (`[data-component-type="columns"]`): `grid-template-columns: 1fr` (single column). The renderer sets `data-component-type` on every component — no `withId()` needed here.
- Dock-bar column (`col-0` slot): `display: none`
- Channel-panel slot: `position: fixed; left: -280px; top: 0; bottom: 0; width: 280px;` — slides in via `left: 0` when `.open` class added. Uses `left` instead of `transform` to avoid creating a new containing block, which would break `position: fixed` modals inside the channel-sidebar Shadow DOM (CSS Transforms Level 1 §6.1).
- Member-panel slot: `position: fixed; right: -280px; top: 0; bottom: 0; width: 280px;` — slides in via `right: 0` when `.open` class added. Same `left`/`right` rationale.
- Chat-area slot: full width, becomes a flex column to accommodate the injected header
- All drag handles: `display: none`
- `will-change: left` / `will-change: right` on drawer slots as rendering hints

### Header bar

Injected by `ResponsiveController` into the chat-area's parent slot container.

```
+----------------------------------+
|  (=)    #general             (o) |  <- 48px header bar
+----------------------------------+
|                                  |
|           messages               |
|                                  |
+----------------------------------+
|  Type a message...               |
+----------------------------------+
```

- Left: hamburger button (`☰` — trigram for heaven) — toggles channel drawer
- Centre: current channel name — updated via `channel-selected` event (falls back to "Chat Demo" when no channel selected)
- Right: people button (`\u{1F465}` — busts in silhouette) — toggles member drawer
- Touch targets: 48x48dp minimum

### Drawer mechanics

- Backdrop: `rgba(0,0,0,0.5)` overlay, controlled by `opacity` + `pointer-events` (no `display` toggle). Clicking it closes the drawer.
- Auto-dismiss: selecting a channel closes the channel drawer.
- Transition: `left 0.3s cubic-bezier(0.4, 0, 0.2, 1)` / `right 0.3s cubic-bezier(0.4, 0, 0.2, 1)` (Material standard easing). Backdrop fades in sync.
- Focus management: when a drawer opens, set `inert` on `[data-component-id="chat-area"]` (the split element, not its slot container — the slot container also holds the injected header bar, which must remain interactive for drawer toggle buttons) and on the opposite drawer's slot container. When the drawer closes, remove `inert`. This traps focus within the open drawer and prevents keyboard interaction with obscured content (WCAG 2.1 SC 2.4.3).
- Escape key: a `keydown` listener (on the mode's `AbortController` signal) closes the active drawer on `Escape`. Follows the WAI-ARIA Dialog Pattern — the drawers are functionally modal (backdrop + `inert`), so Escape should dismiss them. Essential for keyboard-only users who cannot reach the backdrop.

### Channel name in header

The `channel-selected` event payload currently carries only `channelId`. The header needs the channel name. Fix: add `channelName` to the payload in `channel-sidebar.ts`:

```typescript
detail: { topic: "channel-selected", payload: { channelId: id, channelName: name } }
```

One-field addition. The `member-list` and `message-input` components already destructure only the fields they use, so the added field is backward-compatible.

The `ResponsiveController` listens for `channel-selected` in its constructor — not per-mode — and stores `currentChannelName` as instance state. This ensures the phone header displays the correct channel name even when transitioning from desktop/tablet to phone mode after a channel was already selected.

### Accessibility

- `aria-expanded` on toggle buttons
- `aria-hidden="true"` on closed drawers
- Focus moves to drawer on open, returns to trigger button on close
- `inert` attribute on `[data-component-id="chat-area"]` and opposite drawer's slot container when a drawer is open — prevents focus, click, and assistive tech interaction with obscured content. Note: `inert` targets the chat-area component element, not its slot container, because the header bar (injected into the slot container) must remain interactive.
- `Escape` key closes the active drawer (WAI-ARIA Dialog Pattern)
- `@media (prefers-reduced-motion: reduce)`: all transitions `0ms`

## Tablet Layout

One sidebar is always visible. It alternates between channels and members via a tab switcher.

### CSS transformation

- Outer `columns` grid: single column (dock-bar hidden)
- Channel-panel slot: visible by default, `flex: 25`
- Member-panel slot: hidden by default (`display: none`). When active: `style.display = ""` (clears inline override, letting the CSS cascade determine the correct display value — matches the pages-runtime dock-toggle pattern in `site.ts`), `order: -1` repositions to left of chat area
- Chat-area slot: `flex: 75`
- All drag handles: `display: none`

### Tab switcher

Injected by `ResponsiveController` into the active sidebar slot.

```
+----------+-------------------------+
|[Channels | Members]                |
|----------+                         |
| # general|                         |
| # random |       messages          |
| # dev    |                         |
|          +-------------------------+
|          | Type a message...       |
+----------+-------------------------+
```

- Two pill-style tabs styled with `--pages-*` CSS variables
- Compact: 36px height + 8px padding = 52px total
- Switching: toggles `display: none` between slots, moves tab switcher into the newly-visible slot, flips `order`, updates `tabletActiveTab` instance state
- State: `tabletActiveTab` (`'channels' | 'members'`, default `'channels'`) is instance state on the controller — survives mode transitions (desktop→tablet restores the last active tab). Not persisted to URL.

## ResponsiveController

A single class in `responsive.ts`. Created after `loadSite()`.

### Lifecycle

```typescript
const responsive = new ResponsiveController(container);
// on dispose:
responsive.dispose();
```

### Responsibilities

1. **Breakpoint detection** — two `matchMedia` listeners. On change: tear down current mode, set up new mode.
2. **Style injection** — injects one `<style>` element into `document.head`. All responsive CSS scoped under `#app.phone` / `#app.tablet`. Removed on `dispose()`.
3. **Channel tracking** — listens for `channel-selected` in the constructor (not per-mode). Stores `currentChannelName` as instance state. Available to any mode's UI construction — eliminates stale state on viewport transitions.
4. **Phone mode** — creates header bar (using `currentChannelName`), backdrop. Wires toggle buttons, backdrop click, Escape key listener, `inert` management on `[data-component-id="chat-area"]` (not its slot container, which holds the header bar).
5. **Tablet mode** — creates tab switcher using `tabletActiveTab` instance state (defaults to `'channels'`). Wires tab clicks, slot visibility toggling. Tab selection stored as instance state — survives mode transitions.
6. **Desktop mode** — removes all injected elements. Ensures both sidebars visible, `inert` removed from all slots.
7. **Mode teardown** — removes injected DOM, CSS classes, resets inline overrides, removes `inert` from all slots. Uses `AbortController` per mode for clean listener removal.

### Dispose contract

`dispose()` restores the DOM to its pre-controller state:
- Tears down the current mode (removes injected DOM, CSS classes, inline style overrides)
- Removes the injected `<style>` element from `document.head`
- Removes the `phone`/`tablet`/`desktop` CSS class from `#app`
- Aborts both `matchMedia` listeners
- Removes the persistent `channel-selected` listener
- Removes `inert` from all slots

After `dispose()`, the layout renders as the default desktop three-column view. The controller is not reusable after disposal — create a new instance if needed.

**When it's called:** `dispose()` exists for testability (clean teardown between test cases) and future SPA integration (route-level mount/unmount). In the current demo, the app lives for the page lifetime and `dispose()` is not called at runtime.

### DOM discovery

Elements found by `data-component-id` (stable). Slot containers via `closest('[data-slot]')`. Resolved once per mode setup.

### No pages-runtime coupling

The controller operates on rendered DOM only. It does not import pages-runtime APIs or dispatch `pages-dock-toggle` events. Visibility is managed through CSS classes, not the runtime's dock mechanism.

## Transitions & Animations

| What | Property | Duration | Easing | Notes |
|------|----------|----------|--------|-------|
| Phone drawer (channel) | `left` | 300ms | cubic-bezier(0.4, 0, 0.2, 1) | `will-change: left`; uses `left` not `transform` to avoid containing block issue with `position: fixed` modals |
| Phone drawer (member) | `right` | 300ms | cubic-bezier(0.4, 0, 0.2, 1) | `will-change: right`; same rationale |
| Backdrop | `opacity` | 300ms | same | Synced with drawer |
| Tablet tab switch | — | instant | — | No animation |
| Breakpoint change | — | instant | — | Full mode teardown/setup |
| Reduced motion | all | 0ms | — | `prefers-reduced-motion: reduce` |

## Edge Cases

- **Resize with drawer open:** mode teardown closes drawers before new mode sets up.
- **Channel selection across modes:** `selectedChannelId` lives in component instances. Components stay in DOM across modes (repositioned, never destroyed). State survives.
- **WebSocket data during mode change:** events dispatch on `document`, unaffected by layout.
- **Modals (create/delete channel):** `position: fixed` inside Shadow DOM. The phone drawers use `left`/`right` positioning (not `transform`) specifically to preserve `position: fixed` behavior — `transform` would create a containing block that constrains fixed-position descendants to the drawer's 280px width (CSS Transforms Level 1 §6.1).
- **Existing dock-bar URL state:** controller bypasses dock mechanism. No conflict.
- **No swipe gestures:** buttons only. Avoids accessibility issues found in Discord's mobile UX feedback.

## File Changes

| File | Action | Scope |
|------|--------|-------|
| `chat-demo/src/main/webui/src/index.ts` | Modify | Add `withId()` on 3 components. Import + instantiate `ResponsiveController`. |
| `chat-demo/src/main/webui/src/responsive.ts` | New | `ResponsiveController` class (~300-400 lines). |
| `chat-demo/src/main/webui/src/panels/channel-sidebar.ts` | Modify | Add `channelName` to `channel-selected` event payload (one field). |

No changes to pages-runtime or pages-ui.

## Testing Strategy

- **Breakpoint transitions:** mock `matchMedia` to simulate phone/tablet/desktop transitions. Verify mode teardown cleans up all injected elements (header bar, backdrop, tab switcher), CSS classes, and inline style overrides.
- **Drawer state:** verify `inert` is set on `[data-component-id="chat-area"]` (not its slot container) and opposite drawer when a drawer opens, removed when it closes. Verify `aria-expanded` and `aria-hidden` attributes are correct at each state. Verify Escape key closes the active drawer.
- **Channel name persistence:** simulate `channel-selected` event in desktop mode, then trigger phone mode setup. Verify the header displays the stored channel name, not the fallback.
- **Dispose:** call `dispose()`, verify no injected DOM remains, no event listeners leak (AbortController aborted), `#app` has no mode CSS class.
- **Tablet tab switching:** verify toggling between channels/members correctly shows/hides slots and repositions the tab switcher. Verify `tabletActiveTab` state persists across tablet→desktop→tablet transitions.

## Out of Scope

Each out-of-scope item is tracked as a GitHub issue for future planning.

- Swipe-to-reveal gestures — #55. Note: issue #54 body text mentions "swipe/tap-outside" but the acceptance criteria specify only button-triggered panels. The acceptance criteria are authoritative; swipe is aspirational and deferred.
- Emoji palette overflow on narrow screens — #56 (pre-existing, separate concern)
- Platform-level responsive primitives in pages-runtime — #58 (separate epic if needed)
- Touch-specific message interactions (long-press, swipe-to-reply) — #57
