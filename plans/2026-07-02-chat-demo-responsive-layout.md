# Chat-Demo Responsive Layout Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add phone, tablet, and desktop responsive layout modes to the chat-demo UI.

**Architecture:** A `ResponsiveController` class uses `matchMedia` to detect viewport breakpoints, injects a `<style>` element with responsive CSS scoped under `#app.phone` / `#app.tablet`, and injects/removes DOM elements (header bar, backdrop, tab switcher) per mode. The controller operates on the rendered pages-ui DOM via `data-component-id` selectors — no pages-runtime imports.

**Tech Stack:** TypeScript, CSS media queries (via JS `matchMedia`), pages-ui layout DSL (`withId()`), Vitest + happy-dom for testing.

## Global Constraints

- Phone: `< 768px`. Tablet: `768px – 1023px`. Desktop: `>= 1024px`.
- Drawers animate via `left`/`right`, never `transform` (CSS Transforms Level 1 §6.1 — `transform` creates a containing block that breaks `position: fixed` modals in Shadow DOM).
- `inert` attribute targets `[data-component-id="chat-area"]` (the split element), NOT its slot container (the slot container holds the header bar, which must remain interactive).
- All event listeners use `AbortController` signals for clean teardown.
- Touch targets: 48×48dp minimum.
- `@media (prefers-reduced-motion: reduce)`: all transition durations 0ms.
- No changes to pages-runtime or pages-ui packages.

---

### Task 1: Test infrastructure + layout tree IDs + channel event payload

**Files:**
- Modify: `chat-demo/src/main/webui/package.json`
- Create: `chat-demo/src/main/webui/vitest.config.ts`
- Create: `chat-demo/src/main/webui/src/test-helpers.ts`
- Modify: `chat-demo/src/main/webui/src/index.ts`
- Modify: `chat-demo/src/main/webui/src/panels/channel-sidebar.ts`

**Interfaces:**
- Produces: `createMockLayout(): HTMLElement` — builds a mock DOM matching pages-ui's rendered output, used by all subsequent test tasks
- Produces: `mockMatchMedia(initial)` — returns `{ setMatches(query, boolean) }` for simulating breakpoint changes in tests

- [ ] **Step 1: Add vitest + happy-dom to devDependencies**

```bash
cd /Users/mdproctor/claude/casehub/connectors/chat-demo/src/main/webui && npm install --save-dev vitest happy-dom
```

- [ ] **Step 2: Add test script to package.json**

Add to `"scripts"` in `chat-demo/src/main/webui/package.json`:

```json
"test": "vitest run",
"test:watch": "vitest"
```

- [ ] **Step 3: Create vitest.config.ts**

Create `chat-demo/src/main/webui/vitest.config.ts`:

```typescript
import { defineConfig } from "vitest/config";

export default defineConfig({
  test: {
    environment: "happy-dom",
    include: ["src/**/*.test.ts"],
  },
});
```

- [ ] **Step 4: Create test-helpers.ts with mock DOM and matchMedia**

Create `chat-demo/src/main/webui/src/test-helpers.ts`:

```typescript
export function createMockLayout(): HTMLElement {
  document.body.innerHTML = `
    <div id="app">
      <div data-component-type="columns" style="display: grid; grid-template-columns: 0fr 1fr;">
        <div data-slot="col-0">
          <div data-component-type="dock-bar" data-component-id="dock"></div>
        </div>
        <div data-slot="col-1">
          <div data-component-type="split" data-component-id="main-split" style="display: flex; flex-direction: row;">
            <div data-slot="0" style="flex: 20; overflow: hidden;">
              <div data-component-type="host-panel" data-component-id="channel-panel"></div>
            </div>
            <div data-split-handle="0" style="width: 6px;"></div>
            <div data-slot="1" style="flex: 60; overflow: hidden;">
              <div data-component-type="split" data-component-id="chat-area" style="display: flex; flex-direction: column;">
                <div data-slot="0" style="flex: 90; overflow: hidden;"></div>
                <div data-split-handle="0" style="height: 6px;"></div>
                <div data-slot="1" style="flex: 10; overflow: hidden;"></div>
              </div>
            </div>
            <div data-split-handle="1" style="width: 6px;"></div>
            <div data-slot="2" style="flex: 20; overflow: hidden;">
              <div data-component-type="host-panel" data-component-id="member-panel"></div>
            </div>
          </div>
        </div>
      </div>
    </div>
  `;
  return document.getElementById("app")!;
}

export const MQ_TABLET = "(min-width: 768px) and (max-width: 1023px)";
export const MQ_DESKTOP = "(min-width: 1024px)";

interface MockMediaControl {
  setMatches(query: string, matches: boolean): void;
}

export function mockMatchMedia(initial: Record<string, boolean>): MockMediaControl {
  const state = { ...initial };
  const listeners = new Map<string, Set<(e: MediaQueryListEvent) => void>>();

  window.matchMedia = (query: string): MediaQueryList => {
    if (!listeners.has(query)) listeners.set(query, new Set());
    return {
      matches: state[query] ?? false,
      media: query,
      addEventListener(type: string, fn: EventListenerOrEventListenerObject, options?: AddEventListenerOptions | boolean) {
        if (type === "change") {
          const handler = fn as (e: MediaQueryListEvent) => void;
          listeners.get(query)!.add(handler);
          if (typeof options === "object" && options.signal) {
            options.signal.addEventListener("abort", () => {
              listeners.get(query)!.delete(handler);
            });
          }
        }
      },
      removeEventListener(type: string, fn: EventListenerOrEventListenerObject) {
        if (type === "change") listeners.get(query)!.delete(fn as (e: MediaQueryListEvent) => void);
      },
      dispatchEvent: () => true,
      onchange: null,
      addListener: () => {},
      removeListener: () => {},
    } as MediaQueryList;
  };

  return {
    setMatches(query: string, matches: boolean) {
      state[query] = matches;
      const fns = listeners.get(query);
      if (fns) {
        for (const fn of fns) {
          fn({ matches, media: query } as MediaQueryListEvent);
        }
      }
    },
  };
}

export function cleanupDOM(): void {
  document.body.innerHTML = "";
  document.head.querySelectorAll("style[data-responsive]").forEach((el) => el.remove());
}
```

- [ ] **Step 5: Add withId() to layout tree in index.ts**

In `chat-demo/src/main/webui/src/index.ts`, change the `chatApp` declaration from:

```typescript
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
```

to:

```typescript
const chatApp = columns([0, 1],
  [withId("dock", dockBar("vertical", [
    { icon: "\u{1F4AC}", label: "Channels", panelId: "channel-panel", defaultOpen: true },
    { icon: "\u{1F465}", label: "Members", panelId: "member-panel", defaultOpen: true },
  ]))],
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

- [ ] **Step 6: Add channelName to channel-selected event in channel-sidebar.ts**

In `chat-demo/src/main/webui/src/panels/channel-sidebar.ts`, in the `selectChannel` method, change:

```typescript
detail: { topic: "channel-selected", payload: { channelId: id } },
```

to:

```typescript
detail: { topic: "channel-selected", payload: { channelId: id, channelName: this.channels.find((ch) => ch.id === id)?.name ?? "" } },
```

- [ ] **Step 7: Verify typecheck passes**

```bash
cd /Users/mdproctor/claude/casehub/connectors/chat-demo/src/main/webui && npx tsc --noEmit
```

Expected: 0 errors.

- [ ] **Step 8: Verify vitest runs (no tests yet)**

```bash
cd /Users/mdproctor/claude/casehub/connectors/chat-demo/src/main/webui && npx vitest run
```

Expected: "No test files found" or similar — confirms vitest is installed and configured.

- [ ] **Step 9: Commit**

```bash
git add chat-demo/src/main/webui/package.json chat-demo/src/main/webui/package-lock.json chat-demo/src/main/webui/vitest.config.ts chat-demo/src/main/webui/src/test-helpers.ts chat-demo/src/main/webui/src/index.ts chat-demo/src/main/webui/src/panels/channel-sidebar.ts
git commit -m "feat(chat-demo): add stable layout IDs, channelName event payload, and test infrastructure — Refs #54"
```

---

### Task 2: ResponsiveController — core infrastructure

**Files:**
- Create: `chat-demo/src/main/webui/src/responsive.ts`
- Create: `chat-demo/src/main/webui/src/responsive.test.ts`

**Interfaces:**
- Consumes: `createMockLayout()`, `mockMatchMedia()`, `cleanupDOM()`, `MQ_TABLET`, `MQ_DESKTOP` from `test-helpers.ts`
- Produces: `ResponsiveController` class with constructor `(container: HTMLElement)`, `dispose(): void`, and mode switching. Phone/tablet mode setup methods are stubs that add only the CSS class — full implementations in Tasks 3 and 4.

- [ ] **Step 1: Write failing tests for core infrastructure**

Create `chat-demo/src/main/webui/src/responsive.test.ts`:

```typescript
import { describe, it, expect, beforeEach, afterEach } from "vitest";
import { createMockLayout, mockMatchMedia, cleanupDOM, MQ_TABLET, MQ_DESKTOP } from "./test-helpers.js";

// Will fail until ResponsiveController exists
import { ResponsiveController } from "./responsive.js";

describe("ResponsiveController", () => {
  let app: HTMLElement;

  afterEach(() => {
    cleanupDOM();
  });

  describe("CSS injection", () => {
    it("injects a <style> element into document.head on construction", () => {
      mockMatchMedia({ [MQ_DESKTOP]: true, [MQ_TABLET]: false });
      app = createMockLayout();
      const ctrl = new ResponsiveController(app);
      const style = document.head.querySelector("style[data-responsive]");
      expect(style).not.toBeNull();
      expect(style!.textContent).toContain("#app.phone");
      expect(style!.textContent).toContain("#app.tablet");
      ctrl.dispose();
    });

    it("removes the <style> element on dispose", () => {
      mockMatchMedia({ [MQ_DESKTOP]: true, [MQ_TABLET]: false });
      app = createMockLayout();
      const ctrl = new ResponsiveController(app);
      ctrl.dispose();
      const style = document.head.querySelector("style[data-responsive]");
      expect(style).toBeNull();
    });
  });

  describe("mode detection", () => {
    it("sets desktop class when viewport >= 1024px", () => {
      mockMatchMedia({ [MQ_DESKTOP]: true, [MQ_TABLET]: false });
      app = createMockLayout();
      const ctrl = new ResponsiveController(app);
      expect(app.classList.contains("desktop")).toBe(true);
      expect(app.classList.contains("phone")).toBe(false);
      expect(app.classList.contains("tablet")).toBe(false);
      ctrl.dispose();
    });

    it("sets tablet class when viewport 768–1023px", () => {
      mockMatchMedia({ [MQ_DESKTOP]: false, [MQ_TABLET]: true });
      app = createMockLayout();
      const ctrl = new ResponsiveController(app);
      expect(app.classList.contains("tablet")).toBe(true);
      ctrl.dispose();
    });

    it("sets phone class when viewport < 768px", () => {
      mockMatchMedia({ [MQ_DESKTOP]: false, [MQ_TABLET]: false });
      app = createMockLayout();
      const ctrl = new ResponsiveController(app);
      expect(app.classList.contains("phone")).toBe(true);
      ctrl.dispose();
    });
  });

  describe("breakpoint transitions", () => {
    it("switches from desktop to phone on resize", () => {
      const media = mockMatchMedia({ [MQ_DESKTOP]: true, [MQ_TABLET]: false });
      app = createMockLayout();
      const ctrl = new ResponsiveController(app);
      expect(app.classList.contains("desktop")).toBe(true);

      media.setMatches(MQ_DESKTOP, false);
      expect(app.classList.contains("phone")).toBe(true);
      expect(app.classList.contains("desktop")).toBe(false);
      ctrl.dispose();
    });

    it("switches from phone to tablet on resize", () => {
      const media = mockMatchMedia({ [MQ_DESKTOP]: false, [MQ_TABLET]: false });
      app = createMockLayout();
      const ctrl = new ResponsiveController(app);
      expect(app.classList.contains("phone")).toBe(true);

      media.setMatches(MQ_TABLET, true);
      expect(app.classList.contains("tablet")).toBe(true);
      expect(app.classList.contains("phone")).toBe(false);
      ctrl.dispose();
    });
  });

  describe("channel tracking", () => {
    it("stores channelName from channel-selected events", () => {
      mockMatchMedia({ [MQ_DESKTOP]: false, [MQ_TABLET]: false });
      app = createMockLayout();
      const ctrl = new ResponsiveController(app);

      document.dispatchEvent(new CustomEvent("pages-event", {
        bubbles: true,
        composed: true,
        detail: { topic: "channel-selected", payload: { channelId: "ch1", channelName: "general" } },
      }));

      expect((ctrl as unknown as { currentChannelName: string }).currentChannelName).toBe("general");
      ctrl.dispose();
    });
  });

  describe("dispose", () => {
    it("removes mode class from #app", () => {
      mockMatchMedia({ [MQ_DESKTOP]: true, [MQ_TABLET]: false });
      app = createMockLayout();
      const ctrl = new ResponsiveController(app);
      ctrl.dispose();
      expect(app.classList.contains("desktop")).toBe(false);
      expect(app.classList.contains("phone")).toBe(false);
      expect(app.classList.contains("tablet")).toBe(false);
    });

    it("stops responding to breakpoint changes after dispose", () => {
      const media = mockMatchMedia({ [MQ_DESKTOP]: true, [MQ_TABLET]: false });
      app = createMockLayout();
      const ctrl = new ResponsiveController(app);
      ctrl.dispose();

      media.setMatches(MQ_DESKTOP, false);
      expect(app.classList.contains("phone")).toBe(false);
      expect(app.classList.contains("desktop")).toBe(false);
    });
  });
});
```

- [ ] **Step 2: Run tests to verify they fail**

```bash
cd /Users/mdproctor/claude/casehub/connectors/chat-demo/src/main/webui && npx vitest run
```

Expected: FAIL — `Cannot find module './responsive.js'`

- [ ] **Step 3: Implement ResponsiveController core**

Create `chat-demo/src/main/webui/src/responsive.ts`:

```typescript
type Mode = "phone" | "tablet" | "desktop";

const MQ_TABLET = "(min-width: 768px) and (max-width: 1023px)";
const MQ_DESKTOP = "(min-width: 1024px)";

const RESPONSIVE_CSS = `
#app.phone [data-component-type="columns"] {
  grid-template-columns: 1fr !important;
}
#app.phone [data-component-type="columns"] > [data-slot="col-0"] {
  display: none !important;
}
#app.phone [data-split-handle] {
  display: none !important;
}
#app.phone .chat-area-slot {
  display: flex !important;
  flex-direction: column !important;
  flex: 1 !important;
}
#app.phone [data-component-id="chat-area"] {
  flex: 1 !important;
  min-height: 0 !important;
}
#app.phone .channel-drawer-slot {
  position: fixed !important;
  left: -280px;
  top: 0;
  bottom: 0;
  width: 280px !important;
  flex: none !important;
  overflow: visible !important;
  z-index: 50;
  transition: left 0.3s cubic-bezier(0.4, 0, 0.2, 1);
  will-change: left;
  background: var(--pages-bg, #1a1a2e);
}
#app.phone .channel-drawer-slot.open {
  left: 0;
}
#app.phone .member-drawer-slot {
  position: fixed !important;
  right: -280px;
  top: 0;
  bottom: 0;
  width: 280px !important;
  flex: none !important;
  overflow: visible !important;
  z-index: 50;
  transition: right 0.3s cubic-bezier(0.4, 0, 0.2, 1);
  will-change: right;
  background: var(--pages-bg, #1a1a2e);
}
#app.phone .member-drawer-slot.open {
  right: 0;
}
.responsive-backdrop {
  position: fixed;
  inset: 0;
  background: rgba(0, 0, 0, 0.5);
  z-index: 40;
  opacity: 0;
  pointer-events: none;
  transition: opacity 0.3s cubic-bezier(0.4, 0, 0.2, 1);
}
.responsive-backdrop.visible {
  opacity: 1;
  pointer-events: auto;
}
.responsive-header {
  display: flex;
  align-items: center;
  justify-content: space-between;
  height: 48px;
  padding: 0 4px;
  background: var(--pages-bg-alt, #16213e);
  border-bottom: 1px solid var(--pages-border, #3a3a5e);
  flex-shrink: 0;
}
.responsive-header button {
  width: 48px;
  height: 48px;
  display: flex;
  align-items: center;
  justify-content: center;
  background: none;
  border: none;
  color: var(--pages-text, #e0e0e0);
  font-size: 20px;
  cursor: pointer;
  border-radius: var(--pages-radius, 4px);
  transition: background 0.15s;
  -webkit-font-smoothing: antialiased;
}
.responsive-header button:hover {
  background: var(--pages-bg-hover, #1e3a5f);
}
.responsive-header .channel-name {
  font-family: var(--pages-font, system-ui, -apple-system, sans-serif);
  font-size: 15px;
  font-weight: 600;
  color: var(--pages-text, #e0e0e0);
  overflow: hidden;
  text-overflow: ellipsis;
  white-space: nowrap;
}
#app.tablet [data-component-type="columns"] {
  grid-template-columns: 1fr !important;
}
#app.tablet [data-component-type="columns"] > [data-slot="col-0"] {
  display: none !important;
}
#app.tablet [data-split-handle] {
  display: none !important;
}
.responsive-tabs {
  display: flex;
  gap: 4px;
  padding: 8px 8px;
  flex-shrink: 0;
}
.responsive-tabs button {
  flex: 1;
  padding: 6px 12px;
  font-size: 12px;
  font-weight: 600;
  font-family: var(--pages-font, system-ui, -apple-system, sans-serif);
  background: var(--pages-bg, #1a1a2e);
  color: var(--pages-text-muted, #999);
  border: 1px solid var(--pages-border, #3a3a5e);
  border-radius: 16px;
  cursor: pointer;
  transition: all 0.15s;
  -webkit-font-smoothing: antialiased;
}
.responsive-tabs button:hover {
  background: var(--pages-bg-hover, #1e3a5f);
}
.responsive-tabs button.active {
  background: var(--pages-accent, #7c8cf8);
  color: #fff;
  border-color: var(--pages-accent, #7c8cf8);
}
@media (prefers-reduced-motion: reduce) {
  #app.phone .channel-drawer-slot,
  #app.phone .member-drawer-slot,
  .responsive-backdrop {
    transition-duration: 0ms !important;
  }
}
`;

export class ResponsiveController {
  private container: HTMLElement;
  private styleEl: HTMLStyleElement;
  private globalAbort = new AbortController();
  private modeAbort: AbortController | null = null;
  private currentMode: Mode;

  currentChannelName = "Chat Demo";
  tabletActiveTab: "channels" | "members" = "channels";

  private channelSlot: HTMLElement | null = null;
  private memberSlot: HTMLElement | null = null;
  private chatAreaSlot: HTMLElement | null = null;
  private chatAreaEl: HTMLElement | null = null;
  private headerBar: HTMLElement | null = null;
  private backdrop: HTMLElement | null = null;
  private tabSwitcher: HTMLElement | null = null;
  private activeDrawer: "channel" | "member" | null = null;

  constructor(container: HTMLElement) {
    this.container = container;

    this.styleEl = document.createElement("style");
    this.styleEl.dataset.responsive = "";
    this.styleEl.textContent = RESPONSIVE_CSS;
    document.head.appendChild(this.styleEl);

    document.addEventListener("pages-event", this.onChannelSelected, { signal: this.globalAbort.signal });

    const tabletMq = window.matchMedia(MQ_TABLET);
    const desktopMq = window.matchMedia(MQ_DESKTOP);

    this.currentMode = this.detectMode(desktopMq.matches, tabletMq.matches);
    this.applyMode(this.currentMode);

    const onMediaChange = () => {
      const newMode = this.detectMode(desktopMq.matches, tabletMq.matches);
      if (newMode !== this.currentMode) {
        this.teardownMode();
        this.currentMode = newMode;
        this.applyMode(newMode);
      }
    };

    tabletMq.addEventListener("change", onMediaChange, { signal: this.globalAbort.signal });
    desktopMq.addEventListener("change", onMediaChange, { signal: this.globalAbort.signal });
  }

  dispose(): void {
    this.teardownMode();
    this.globalAbort.abort();
    this.styleEl.remove();
    this.container.classList.remove("phone", "tablet", "desktop");
  }

  private onChannelSelected = (e: Event): void => {
    const { topic, payload } = (e as CustomEvent).detail;
    if (topic !== "channel-selected") return;
    const { channelName } = payload as { channelId: string; channelName: string };
    if (channelName) this.currentChannelName = channelName;

    const nameEl = this.headerBar?.querySelector(".channel-name");
    if (nameEl) nameEl.textContent = `#${channelName}`;

    if (this.currentMode === "phone" && this.activeDrawer === "channel") {
      this.closeDrawer();
    }
  };

  private detectMode(desktop: boolean, tablet: boolean): Mode {
    if (desktop) return "desktop";
    if (tablet) return "tablet";
    return "phone";
  }

  private resolveSlots(): void {
    const channelPanelEl = this.container.querySelector<HTMLElement>('[data-component-id="channel-panel"]');
    const memberPanelEl = this.container.querySelector<HTMLElement>('[data-component-id="member-panel"]');
    this.chatAreaEl = this.container.querySelector<HTMLElement>('[data-component-id="chat-area"]');
    this.channelSlot = channelPanelEl?.closest<HTMLElement>("[data-slot]") ?? null;
    this.memberSlot = memberPanelEl?.closest<HTMLElement>("[data-slot]") ?? null;
    this.chatAreaSlot = this.chatAreaEl?.closest<HTMLElement>("[data-slot]") ?? null;
  }

  private applyMode(mode: Mode): void {
    this.container.classList.remove("phone", "tablet", "desktop");
    this.container.classList.add(mode);
    this.resolveSlots();
    this.modeAbort = new AbortController();

    switch (mode) {
      case "phone":
        this.setupPhone();
        break;
      case "tablet":
        this.setupTablet();
        break;
      case "desktop":
        this.setupDesktop();
        break;
    }
  }

  private teardownMode(): void {
    this.modeAbort?.abort();
    this.modeAbort = null;

    this.headerBar?.remove();
    this.headerBar = null;
    this.backdrop?.remove();
    this.backdrop = null;
    this.tabSwitcher?.remove();
    this.tabSwitcher = null;
    this.activeDrawer = null;

    this.channelSlot?.classList.remove("channel-drawer-slot", "open");
    this.memberSlot?.classList.remove("member-drawer-slot", "open");
    this.chatAreaSlot?.classList.remove("chat-area-slot");

    this.channelSlot?.removeAttribute("aria-hidden");
    this.memberSlot?.removeAttribute("aria-hidden");
    this.chatAreaEl?.removeAttribute("inert");
    this.channelSlot?.removeAttribute("inert");
    this.memberSlot?.removeAttribute("inert");

    if (this.channelSlot) this.channelSlot.style.display = "";
    if (this.memberSlot) {
      this.memberSlot.style.display = "";
      this.memberSlot.style.order = "";
    }
    if (this.chatAreaSlot) this.chatAreaSlot.style.flex = "";
  }

  private setupPhone(): void {
    if (!this.channelSlot || !this.memberSlot || !this.chatAreaSlot || !this.chatAreaEl) return;

    this.channelSlot.classList.add("channel-drawer-slot");
    this.memberSlot.classList.add("member-drawer-slot");
    this.chatAreaSlot.classList.add("chat-area-slot");

    this.channelSlot.setAttribute("aria-hidden", "true");
    this.memberSlot.setAttribute("aria-hidden", "true");

    this.headerBar = document.createElement("div");
    this.headerBar.className = "responsive-header";
    const menuBtn = document.createElement("button");
    menuBtn.textContent = "☰";
    menuBtn.title = "Channels";
    menuBtn.setAttribute("aria-expanded", "false");
    const nameSpan = document.createElement("span");
    nameSpan.className = "channel-name";
    nameSpan.textContent = `#${this.currentChannelName === "Chat Demo" ? "" : this.currentChannelName}` || "Chat Demo";
    if (this.currentChannelName && this.currentChannelName !== "Chat Demo") {
      nameSpan.textContent = `#${this.currentChannelName}`;
    } else {
      nameSpan.textContent = "Chat Demo";
    }
    const membersBtn = document.createElement("button");
    membersBtn.textContent = "\u{1F465}";
    membersBtn.title = "Members";
    membersBtn.setAttribute("aria-expanded", "false");
    this.headerBar.append(menuBtn, nameSpan, membersBtn);
    this.chatAreaSlot.insertBefore(this.headerBar, this.chatAreaSlot.firstChild);

    this.backdrop = document.createElement("div");
    this.backdrop.className = "responsive-backdrop";
    this.container.appendChild(this.backdrop);

    const signal = this.modeAbort!.signal;

    menuBtn.addEventListener("click", () => {
      if (this.activeDrawer === "channel") this.closeDrawer();
      else this.openDrawer("channel");
    }, { signal });

    membersBtn.addEventListener("click", () => {
      if (this.activeDrawer === "member") this.closeDrawer();
      else this.openDrawer("member");
    }, { signal });

    this.backdrop.addEventListener("click", () => this.closeDrawer(), { signal });

    document.addEventListener("keydown", (e) => {
      if (e.key === "Escape" && this.activeDrawer) this.closeDrawer();
    }, { signal });
  }

  private openDrawer(side: "channel" | "member"): void {
    if (this.activeDrawer) this.closeDrawer();
    this.activeDrawer = side;

    const drawerSlot = side === "channel" ? this.channelSlot : this.memberSlot;
    const oppositeSlot = side === "channel" ? this.memberSlot : this.channelSlot;
    drawerSlot?.classList.add("open");
    drawerSlot?.setAttribute("aria-hidden", "false");
    this.backdrop?.classList.add("visible");

    this.chatAreaEl?.setAttribute("inert", "");
    oppositeSlot?.setAttribute("inert", "");

    const buttons = this.headerBar?.querySelectorAll("button");
    if (buttons) {
      const btnIndex = side === "channel" ? 0 : 1;
      buttons[btnIndex]?.setAttribute("aria-expanded", "true");
    }

    drawerSlot?.focus();
  }

  private closeDrawer(): void {
    if (!this.activeDrawer) return;
    const side = this.activeDrawer;
    this.activeDrawer = null;

    const drawerSlot = side === "channel" ? this.channelSlot : this.memberSlot;
    drawerSlot?.classList.remove("open");
    drawerSlot?.setAttribute("aria-hidden", "true");
    this.backdrop?.classList.remove("visible");

    this.chatAreaEl?.removeAttribute("inert");
    this.channelSlot?.removeAttribute("inert");
    this.memberSlot?.removeAttribute("inert");

    const buttons = this.headerBar?.querySelectorAll("button");
    if (buttons) {
      buttons[0]?.setAttribute("aria-expanded", "false");
      buttons[1]?.setAttribute("aria-expanded", "false");
    }

    const btnIndex = side === "channel" ? 0 : 1;
    buttons?.[btnIndex]?.focus();
  }

  private setupTablet(): void {
    if (!this.channelSlot || !this.memberSlot || !this.chatAreaSlot) return;

    this.chatAreaSlot.style.flex = "75";

    this.tabSwitcher = document.createElement("div");
    this.tabSwitcher.className = "responsive-tabs";
    const channelsTab = document.createElement("button");
    channelsTab.textContent = "Channels";
    channelsTab.dataset.tab = "channels";
    const membersTab = document.createElement("button");
    membersTab.textContent = "Members";
    membersTab.dataset.tab = "members";
    this.tabSwitcher.append(channelsTab, membersTab);

    this.switchTabTo(this.tabletActiveTab);

    const signal = this.modeAbort!.signal;
    channelsTab.addEventListener("click", () => this.switchTab("channels"), { signal });
    membersTab.addEventListener("click", () => this.switchTab("members"), { signal });
  }

  private switchTab(tab: "channels" | "members"): void {
    this.tabletActiveTab = tab;
    this.switchTabTo(tab);
  }

  private switchTabTo(tab: "channels" | "members"): void {
    if (!this.channelSlot || !this.memberSlot || !this.tabSwitcher) return;

    if (tab === "channels") {
      this.channelSlot.style.display = "";
      this.channelSlot.style.flex = "25";
      this.memberSlot.style.display = "none";
      this.memberSlot.style.order = "";
      this.channelSlot.insertBefore(this.tabSwitcher, this.channelSlot.firstChild);
    } else {
      this.channelSlot.style.display = "none";
      this.memberSlot.style.display = "";
      this.memberSlot.style.flex = "25";
      this.memberSlot.style.order = "-1";
      this.memberSlot.insertBefore(this.tabSwitcher, this.memberSlot.firstChild);
    }

    const buttons = this.tabSwitcher.querySelectorAll("button");
    buttons.forEach((btn) => {
      btn.classList.toggle("active", btn.dataset.tab === tab);
    });
  }

  private setupDesktop(): void {
    if (this.channelSlot) this.channelSlot.style.display = "";
    if (this.memberSlot) {
      this.memberSlot.style.display = "";
      this.memberSlot.style.order = "";
    }
  }
}
```

- [ ] **Step 4: Run tests to verify they pass**

```bash
cd /Users/mdproctor/claude/casehub/connectors/chat-demo/src/main/webui && npx vitest run
```

Expected: all tests PASS.

- [ ] **Step 5: Commit**

```bash
git add chat-demo/src/main/webui/src/responsive.ts chat-demo/src/main/webui/src/responsive.test.ts
git commit -m "feat(chat-demo): ResponsiveController core — CSS injection, breakpoint detection, dispose — Refs #54"
```

---

### Task 3: ResponsiveController — phone mode tests

**Files:**
- Modify: `chat-demo/src/main/webui/src/responsive.test.ts`

**Interfaces:**
- Consumes: `ResponsiveController` from `responsive.ts`, test helpers from `test-helpers.ts`
- Produces: test coverage for phone mode (drawers, header, inert, escape, auto-dismiss)

- [ ] **Step 1: Write phone mode tests**

Append to `chat-demo/src/main/webui/src/responsive.test.ts`:

```typescript
describe("phone mode", () => {
  let media: ReturnType<typeof mockMatchMedia>;

  beforeEach(() => {
    media = mockMatchMedia({ [MQ_DESKTOP]: false, [MQ_TABLET]: false });
    app = createMockLayout();
  });

  it("injects header bar into chat-area slot", () => {
    const ctrl = new ResponsiveController(app);
    const header = app.querySelector(".responsive-header");
    expect(header).not.toBeNull();
    expect(header!.querySelector(".channel-name")!.textContent).toBe("Chat Demo");
    ctrl.dispose();
  });

  it("injects backdrop into container", () => {
    const ctrl = new ResponsiveController(app);
    expect(app.querySelector(".responsive-backdrop")).not.toBeNull();
    ctrl.dispose();
  });

  it("adds drawer CSS classes to sidebar slots", () => {
    const ctrl = new ResponsiveController(app);
    const channelSlot = app.querySelector('[data-component-id="channel-panel"]')!.closest("[data-slot]")!;
    const memberSlot = app.querySelector('[data-component-id="member-panel"]')!.closest("[data-slot]")!;
    expect(channelSlot.classList.contains("channel-drawer-slot")).toBe(true);
    expect(memberSlot.classList.contains("member-drawer-slot")).toBe(true);
    ctrl.dispose();
  });

  it("opens channel drawer on hamburger click", () => {
    const ctrl = new ResponsiveController(app);
    const menuBtn = app.querySelector(".responsive-header button")!;
    (menuBtn as HTMLElement).click();
    const channelSlot = app.querySelector('[data-component-id="channel-panel"]')!.closest("[data-slot]")!;
    expect(channelSlot.classList.contains("open")).toBe(true);
    expect(app.querySelector(".responsive-backdrop")!.classList.contains("visible")).toBe(true);
    ctrl.dispose();
  });

  it("sets inert on chat-area element (not slot) when drawer opens", () => {
    const ctrl = new ResponsiveController(app);
    const menuBtn = app.querySelector(".responsive-header button")!;
    (menuBtn as HTMLElement).click();
    const chatArea = app.querySelector('[data-component-id="chat-area"]')!;
    expect(chatArea.hasAttribute("inert")).toBe(true);
    const chatAreaSlot = chatArea.closest("[data-slot]")!;
    expect(chatAreaSlot.hasAttribute("inert")).toBe(false);
    ctrl.dispose();
  });

  it("sets inert on opposite drawer slot when drawer opens", () => {
    const ctrl = new ResponsiveController(app);
    const menuBtn = app.querySelector(".responsive-header button")!;
    (menuBtn as HTMLElement).click();
    const memberSlot = app.querySelector('[data-component-id="member-panel"]')!.closest("[data-slot]")!;
    expect(memberSlot.hasAttribute("inert")).toBe(true);
    ctrl.dispose();
  });

  it("closes drawer on backdrop click", () => {
    const ctrl = new ResponsiveController(app);
    const menuBtn = app.querySelector(".responsive-header button")!;
    (menuBtn as HTMLElement).click();
    const backdrop = app.querySelector(".responsive-backdrop")! as HTMLElement;
    backdrop.click();
    const channelSlot = app.querySelector('[data-component-id="channel-panel"]')!.closest("[data-slot]")!;
    expect(channelSlot.classList.contains("open")).toBe(false);
    expect(backdrop.classList.contains("visible")).toBe(false);
    ctrl.dispose();
  });

  it("closes drawer on Escape key", () => {
    const ctrl = new ResponsiveController(app);
    const menuBtn = app.querySelector(".responsive-header button")!;
    (menuBtn as HTMLElement).click();
    document.dispatchEvent(new KeyboardEvent("keydown", { key: "Escape" }));
    const channelSlot = app.querySelector('[data-component-id="channel-panel"]')!.closest("[data-slot]")!;
    expect(channelSlot.classList.contains("open")).toBe(false);
    ctrl.dispose();
  });

  it("removes inert from all elements when drawer closes", () => {
    const ctrl = new ResponsiveController(app);
    const menuBtn = app.querySelector(".responsive-header button")!;
    (menuBtn as HTMLElement).click();
    document.dispatchEvent(new KeyboardEvent("keydown", { key: "Escape" }));
    const chatArea = app.querySelector('[data-component-id="chat-area"]')!;
    const memberSlot = app.querySelector('[data-component-id="member-panel"]')!.closest("[data-slot]")!;
    expect(chatArea.hasAttribute("inert")).toBe(false);
    expect(memberSlot.hasAttribute("inert")).toBe(false);
    ctrl.dispose();
  });

  it("sets aria-expanded on toggle buttons", () => {
    const ctrl = new ResponsiveController(app);
    const buttons = app.querySelectorAll(".responsive-header button");
    expect(buttons[0]!.getAttribute("aria-expanded")).toBe("false");

    (buttons[0] as HTMLElement).click();
    expect(buttons[0]!.getAttribute("aria-expanded")).toBe("true");

    document.dispatchEvent(new KeyboardEvent("keydown", { key: "Escape" }));
    expect(buttons[0]!.getAttribute("aria-expanded")).toBe("false");
    ctrl.dispose();
  });

  it("auto-closes channel drawer on channel selection", () => {
    const ctrl = new ResponsiveController(app);
    const menuBtn = app.querySelector(".responsive-header button")!;
    (menuBtn as HTMLElement).click();

    document.dispatchEvent(new CustomEvent("pages-event", {
      bubbles: true, composed: true,
      detail: { topic: "channel-selected", payload: { channelId: "ch2", channelName: "random" } },
    }));

    const channelSlot = app.querySelector('[data-component-id="channel-panel"]')!.closest("[data-slot]")!;
    expect(channelSlot.classList.contains("open")).toBe(false);
    ctrl.dispose();
  });

  it("updates header channel name on channel selection", () => {
    const ctrl = new ResponsiveController(app);
    document.dispatchEvent(new CustomEvent("pages-event", {
      bubbles: true, composed: true,
      detail: { topic: "channel-selected", payload: { channelId: "ch1", channelName: "general" } },
    }));
    expect(app.querySelector(".channel-name")!.textContent).toBe("#general");
    ctrl.dispose();
  });

  it("uses stored channelName when entering phone mode after desktop selection", () => {
    const desktopMedia = mockMatchMedia({ [MQ_DESKTOP]: true, [MQ_TABLET]: false });
    app = createMockLayout();
    const ctrl = new ResponsiveController(app);

    document.dispatchEvent(new CustomEvent("pages-event", {
      bubbles: true, composed: true,
      detail: { topic: "channel-selected", payload: { channelId: "ch3", channelName: "dev" } },
    }));

    desktopMedia.setMatches(MQ_DESKTOP, false);
    expect(app.querySelector(".channel-name")!.textContent).toBe("#dev");
    ctrl.dispose();
  });

  it("cleans up phone DOM elements on mode transition", () => {
    const ctrl = new ResponsiveController(app);
    expect(app.querySelector(".responsive-header")).not.toBeNull();
    expect(app.querySelector(".responsive-backdrop")).not.toBeNull();

    media.setMatches(MQ_DESKTOP, true);
    expect(app.querySelector(".responsive-header")).toBeNull();
    expect(app.querySelector(".responsive-backdrop")).toBeNull();
    ctrl.dispose();
  });
});
```

- [ ] **Step 2: Run tests to verify they pass**

```bash
cd /Users/mdproctor/claude/casehub/connectors/chat-demo/src/main/webui && npx vitest run
```

Expected: all tests PASS (the implementation from Task 2 already includes phone mode).

- [ ] **Step 3: Commit**

```bash
git add chat-demo/src/main/webui/src/responsive.test.ts
git commit -m "test(chat-demo): phone mode tests — drawers, inert, escape, auto-dismiss — Refs #54"
```

---

### Task 4: ResponsiveController — tablet mode tests + integration

**Files:**
- Modify: `chat-demo/src/main/webui/src/responsive.test.ts`
- Modify: `chat-demo/src/main/webui/src/index.ts`

**Interfaces:**
- Consumes: `ResponsiveController` from `responsive.ts`, test helpers from `test-helpers.ts`
- Produces: test coverage for tablet mode; wired-up integration in `index.ts`

- [ ] **Step 1: Write tablet mode tests**

Append to `chat-demo/src/main/webui/src/responsive.test.ts`:

```typescript
describe("tablet mode", () => {
  let media: ReturnType<typeof mockMatchMedia>;

  beforeEach(() => {
    media = mockMatchMedia({ [MQ_DESKTOP]: false, [MQ_TABLET]: true });
    app = createMockLayout();
  });

  it("injects tab switcher into channel-panel slot", () => {
    const ctrl = new ResponsiveController(app);
    const channelSlot = app.querySelector('[data-component-id="channel-panel"]')!.closest("[data-slot]")!;
    expect(channelSlot.querySelector(".responsive-tabs")).not.toBeNull();
    ctrl.dispose();
  });

  it("shows channels tab as active by default", () => {
    const ctrl = new ResponsiveController(app);
    const tabs = app.querySelectorAll(".responsive-tabs button");
    expect(tabs[0]!.classList.contains("active")).toBe(true);
    expect(tabs[1]!.classList.contains("active")).toBe(false);
    ctrl.dispose();
  });

  it("shows channel-panel slot and hides member-panel slot by default", () => {
    const ctrl = new ResponsiveController(app);
    const channelSlot = app.querySelector('[data-component-id="channel-panel"]')!.closest("[data-slot]") as HTMLElement;
    const memberSlot = app.querySelector('[data-component-id="member-panel"]')!.closest("[data-slot]") as HTMLElement;
    expect(channelSlot.style.display).not.toBe("none");
    expect(memberSlot.style.display).toBe("none");
    ctrl.dispose();
  });

  it("switches to members tab on click", () => {
    const ctrl = new ResponsiveController(app);
    const membersTab = app.querySelector('.responsive-tabs button[data-tab="members"]')! as HTMLElement;
    membersTab.click();

    const channelSlot = app.querySelector('[data-component-id="channel-panel"]')!.closest("[data-slot]") as HTMLElement;
    const memberSlot = app.querySelector('[data-component-id="member-panel"]')!.closest("[data-slot]") as HTMLElement;
    expect(channelSlot.style.display).toBe("none");
    expect(memberSlot.style.display).not.toBe("none");
    expect(memberSlot.style.order).toBe("-1");
    ctrl.dispose();
  });

  it("moves tab switcher into member-panel slot when members tab active", () => {
    const ctrl = new ResponsiveController(app);
    const membersTab = app.querySelector('.responsive-tabs button[data-tab="members"]')! as HTMLElement;
    membersTab.click();

    const memberSlot = app.querySelector('[data-component-id="member-panel"]')!.closest("[data-slot]")!;
    expect(memberSlot.querySelector(".responsive-tabs")).not.toBeNull();
    ctrl.dispose();
  });

  it("preserves tabletActiveTab across mode transitions", () => {
    const ctrl = new ResponsiveController(app);
    const membersTab = app.querySelector('.responsive-tabs button[data-tab="members"]')! as HTMLElement;
    membersTab.click();

    media.setMatches(MQ_TABLET, false);
    media.setMatches(MQ_DESKTOP, true);
    media.setMatches(MQ_DESKTOP, false);
    media.setMatches(MQ_TABLET, true);

    const memberSlot = app.querySelector('[data-component-id="member-panel"]')!.closest("[data-slot]") as HTMLElement;
    expect(memberSlot.style.display).not.toBe("none");
    expect(memberSlot.style.order).toBe("-1");

    const tabs = app.querySelectorAll(".responsive-tabs button");
    expect(tabs[1]!.classList.contains("active")).toBe(true);
    ctrl.dispose();
  });

  it("sets chat-area slot flex to 75", () => {
    const ctrl = new ResponsiveController(app);
    const chatAreaSlot = app.querySelector('[data-component-id="chat-area"]')!.closest("[data-slot]") as HTMLElement;
    expect(chatAreaSlot.style.flex).toBe("75");
    ctrl.dispose();
  });

  it("cleans up tablet DOM on mode transition", () => {
    const ctrl = new ResponsiveController(app);
    expect(app.querySelector(".responsive-tabs")).not.toBeNull();

    media.setMatches(MQ_TABLET, false);
    media.setMatches(MQ_DESKTOP, true);
    expect(app.querySelector(".responsive-tabs")).toBeNull();
    ctrl.dispose();
  });
});
```

- [ ] **Step 2: Run tests to verify they pass**

```bash
cd /Users/mdproctor/claude/casehub/connectors/chat-demo/src/main/webui && npx vitest run
```

Expected: all tests PASS.

- [ ] **Step 3: Wire ResponsiveController in index.ts**

In `chat-demo/src/main/webui/src/index.ts`, add the import at the top:

```typescript
import { ResponsiveController } from "./responsive.js";
```

Then inside the `.then((site) => { ... })` callback, after `site.setTheme("dark");`, add:

```typescript
    const responsive = new ResponsiveController(container);
```

The `responsive` variable is intentionally unused after construction — the controller manages its own lifecycle via `matchMedia` listeners and `AbortController`. No `dispose()` call needed at runtime.

- [ ] **Step 4: Run typecheck**

```bash
cd /Users/mdproctor/claude/casehub/connectors/chat-demo/src/main/webui && npx tsc --noEmit
```

Expected: 0 errors.

- [ ] **Step 5: Build the project**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn clean install -Pdemo -Pui
```

Expected: BUILD SUCCESS.

- [ ] **Step 6: Commit**

```bash
git add chat-demo/src/main/webui/src/responsive.test.ts chat-demo/src/main/webui/src/index.ts
git commit -m "feat(chat-demo): responsive layout — phone drawers, tablet tabs, desktop unchanged — Closes #54"
```

---

## Post-Implementation Checklist

- [ ] Run all tests: `cd chat-demo/src/main/webui && npx vitest run`
- [ ] Run typecheck: `cd chat-demo/src/main/webui && npx tsc --noEmit`
- [ ] Full build: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn clean install -Pdemo -Pui`
- [ ] Manual browser verification:
  - Desktop (>= 1024px): three-column layout unchanged
  - Tablet (768–1023px): single sidebar with Channels/Members tabs, chat area fills remaining space
  - Phone (< 768px): full-screen chat with header bar, hamburger opens channel drawer from left, people icon opens member drawer from right, backdrop dismisses, Escape key dismisses, channel selection auto-dismisses
  - Resize transitions: desktop → phone → tablet → desktop (verify no stale DOM, no leaked classes)
- [ ] Invoke `superpowers:requesting-code-review`
- [ ] Invoke `implementation-doc-sync`
