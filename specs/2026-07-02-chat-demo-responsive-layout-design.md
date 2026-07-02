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

- Outer `columns` grid: `grid-template-columns: 1fr` (single column)
- Dock-bar column (`col-0` slot): `display: none`
- Channel-panel slot: `position: fixed; left: 0; top: 0; bottom: 0; width: 280px; transform: translateX(-100%);` — slides in when `.open` class added
- Member-panel slot: `position: fixed; right: 0; top: 0; bottom: 0; width: 280px; transform: translateX(100%);` — slides in when `.open` class added
- Chat-area slot: full width, becomes a flex column to accommodate the injected header
- All drag handles: `display: none`

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
- Transition: `transform 0.3s cubic-bezier(0.4, 0, 0.2, 1)` (Material standard easing). Backdrop fades in sync.

### Channel name in header

The `channel-selected` event payload currently carries only `channelId`. The header needs the channel name. Fix: add `channelName` to the payload in `channel-sidebar.ts`:

```typescript
detail: { topic: "channel-selected", payload: { channelId: id, channelName: name } }
```

One-field addition. The `member-list` and `message-input` components already destructure only the fields they use, so the added field is backward-compatible.

### Accessibility

- `aria-expanded` on toggle buttons
- `aria-hidden` on closed drawers
- Focus moves to drawer on open, returns to trigger button on close
- `@media (prefers-reduced-motion: reduce)`: all transitions `0ms`

## Tablet Layout

One sidebar is always visible. It alternates between channels and members via a tab switcher.

### CSS transformation

- Outer `columns` grid: single column (dock-bar hidden)
- Channel-panel slot: visible by default, `flex: 25`
- Member-panel slot: hidden by default (`display: none`). When active: `display` restored, `order: -1` repositions to left of chat area
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
- Switching: toggles `display: none` between slots, moves tab switcher into the newly-visible slot, flips `order`
- State is local to the controller — not persisted to URL

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
3. **Phone mode** — creates header bar, backdrop. Wires toggle buttons, backdrop click, `channel-selected` listener.
4. **Tablet mode** — creates tab switcher. Wires tab clicks, slot visibility toggling.
5. **Desktop mode** — removes all injected elements. Ensures both sidebars visible.
6. **Mode teardown** — removes injected DOM, CSS classes, resets inline overrides. Uses `AbortController` per mode for clean listener removal.

### DOM discovery

Elements found by `data-component-id` (stable). Slot containers via `closest('[data-slot]')`. Resolved once per mode setup.

### No pages-runtime coupling

The controller operates on rendered DOM only. It does not import pages-runtime APIs or dispatch `pages-dock-toggle` events. Visibility is managed through CSS classes, not the runtime's dock mechanism.

## Transitions & Animations

| What | Property | Duration | Easing | Notes |
|------|----------|----------|--------|-------|
| Phone drawer | `transform` | 300ms | cubic-bezier(0.4, 0, 0.2, 1) | Never animate width/left/display |
| Backdrop | `opacity` | 300ms | same | Synced with drawer |
| Tablet tab switch | — | instant | — | No animation |
| Breakpoint change | — | instant | — | Full mode teardown/setup |
| Reduced motion | all | 0ms | — | `prefers-reduced-motion: reduce` |

## Edge Cases

- **Resize with drawer open:** mode teardown closes drawers before new mode sets up.
- **Channel selection across modes:** `selectedChannelId` lives in component instances. Components stay in DOM across modes (repositioned, never destroyed). State survives.
- **WebSocket data during mode change:** events dispatch on `document`, unaffected by layout.
- **Modals (create/delete channel):** `position: fixed` inside Shadow DOM. Work at all viewport sizes.
- **Existing dock-bar URL state:** controller bypasses dock mechanism. No conflict.
- **No swipe gestures:** buttons only. Avoids accessibility issues found in Discord's mobile UX feedback.

## File Changes

| File | Action | Scope |
|------|--------|-------|
| `chat-demo/src/main/webui/src/index.ts` | Modify | Add `withId()` on 3 components. Import + instantiate `ResponsiveController`. |
| `chat-demo/src/main/webui/src/responsive.ts` | New | `ResponsiveController` class (~300-400 lines). |
| `chat-demo/src/main/webui/src/panels/channel-sidebar.ts` | Modify | Add `channelName` to `channel-selected` event payload (one field). |

No changes to pages-runtime or pages-ui.

## Out of Scope

- Swipe-to-reveal gestures (can be added later)
- Emoji palette overflow on narrow screens (pre-existing, separate issue)
- Platform-level responsive primitives in pages-runtime (separate epic if needed)
- Touch-specific message interactions (long-press, swipe-to-reply)
