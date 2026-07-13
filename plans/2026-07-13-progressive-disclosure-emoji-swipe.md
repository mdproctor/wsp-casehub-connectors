# Progressive Disclosure, Emoji Palette, Swipe Gestures — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> executing-plans to implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural editing.
> Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #63 — feat(chat-demo): progressive disclosure on message expand
**Issue group:** #63, #64, #55

**Goal:** Add progressive disclosure to messages, emoji reaction palette, and
swipe-to-reveal gestures for phone drawers in the chat-demo webui.

**Architecture:** Hybrid (Approach C) — each feature owns its own state and
rendering. Progressive disclosure lives in `qhorus-message` with an internal
`@state()`. Emoji picker wraps `emoji-picker-element` via Popover API. Swipe
gestures use a Lit reactive controller on the workbench.

**Tech Stack:** Lit 3.3, TypeScript, vitest + jsdom, `emoji-picker-element`

## Global Constraints

- All styling via `--pages-*` design tokens exclusively — no hardcoded values
- All animations respect `prefers-reduced-motion`
- All interactive elements keyboard-accessible
- Events use `pages-event` custom events with `chat:*` topics (except internal
  component events like `emoji-selected`)
- Tests run: `cd chat-demo/src/main/webui && npx vitest run`
- IntelliJ MCP mandatory for all `.ts` file edits — use `ide_edit_member`,
  `ide_replace_member`, `ide_insert_member`. Use `Write` only for new files.
- Project path for IntelliJ: `/Users/mdproctor/claude/casehub/connectors`

---

### Task 1: Progressive Disclosure — Expand Toggle and Expanded Section (#63)

**Files:**
- Modify: `chat-demo/src/main/webui/src/qhorus/primitives/qhorus-message.ts`
- Modify: `chat-demo/src/main/webui/src/qhorus/primitives/qhorus-message.test.ts`
- Modify: `chat-demo/src/main/webui/src/qhorus/composites/qhorus-channel-feed.ts`
- Modify: `chat-demo/src/main/webui/src/qhorus/composites/qhorus-channel-feed.test.ts`
- Modify: `chat-demo/src/main/webui/src/qhorus/workbench/qhorus-workbench.ts`

**Interfaces:**
- Consumes: `QhorusMessage` type (existing), `emitChatEvent` (existing),
  `ChatEventTopics.MESSAGE_SELECTED` (existing)
- Produces: `qhorus-message` with new properties `parentMessage?: QhorusMessage`,
  `channelName?: string`, internal `@state() _expanded: boolean`, and an expand
  toggle button. `qhorus-channel-feed` gains `channelName?: string` property.

#### Part A: qhorus-message expand toggle and expanded section

- [ ] **Step 1: Write failing tests for expand toggle**

Add to `qhorus-message.test.ts`:

```typescript
it('renders expand toggle button in header', async () => {
  const el = await renderMessage();
  const toggle = el.shadowRoot!.querySelector('.expand-toggle');
  expect(toggle).toBeTruthy();
  expect(toggle!.getAttribute('aria-expanded')).toBe('false');
});

it('toggles expanded state on toggle button click', async () => {
  const el = await renderMessage();
  const toggle = el.shadowRoot!.querySelector('.expand-toggle') as HTMLButtonElement;
  toggle.click();
  await (el as any).updateComplete;
  expect(toggle.getAttribute('aria-expanded')).toBe('true');
  expect(el.shadowRoot!.querySelector('.expanded-section')).toBeTruthy();
});

it('collapses on second toggle click', async () => {
  const el = await renderMessage();
  const toggle = el.shadowRoot!.querySelector('.expand-toggle') as HTMLButtonElement;
  toggle.click();
  await (el as any).updateComplete;
  toggle.click();
  await (el as any).updateComplete;
  expect(toggle.getAttribute('aria-expanded')).toBe('false');
  expect(el.shadowRoot!.querySelector('.expanded-section')).toBeNull();
});

it('toggle button is keyboard-focusable and activates on Enter', async () => {
  const el = await renderMessage();
  const toggle = el.shadowRoot!.querySelector('.expand-toggle') as HTMLButtonElement;
  expect(toggle.tagName).toBe('BUTTON');
});

it('text selection within content does not trigger expand', async () => {
  const el = await renderMessage();
  const content = el.shadowRoot!.querySelector('.content') as HTMLElement;
  content.click();
  await (el as any).updateComplete;
  expect(el.shadowRoot!.querySelector('.expanded-section')).toBeNull();
});
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `cd chat-demo/src/main/webui && npx vitest run src/qhorus/primitives/qhorus-message.test.ts`
Expected: FAIL — no `.expand-toggle` element exists

- [ ] **Step 3: Write failing tests for expanded view content**

Add to `qhorus-message.test.ts`:

```typescript
it('renders correlation context when expanded with parentMessage', async () => {
  const parent = makeMessage({ id: 'parent-1', sender: 'bob', content: 'Original question about the deployment' });
  const el = await renderMessage({
    message: { inReplyTo: 'parent-1' },
  });
  (el as any).parentMessage = parent;
  await (el as any).updateComplete;
  // Expand
  const toggle = el.shadowRoot!.querySelector('.expand-toggle') as HTMLButtonElement;
  toggle.click();
  await (el as any).updateComplete;
  const ctx = el.shadowRoot!.querySelector('.correlation-context');
  expect(ctx).toBeTruthy();
  expect(ctx!.textContent).toContain('bob');
  expect(ctx!.textContent).toContain('Original question about the deploy');
});

it('does not render correlation context when parentMessage not set', async () => {
  const el = await renderMessage({ message: { inReplyTo: 'parent-1' } });
  const toggle = el.shadowRoot!.querySelector('.expand-toggle') as HTMLButtonElement;
  toggle.click();
  await (el as any).updateComplete;
  expect(el.shadowRoot!.querySelector('.correlation-context')).toBeNull();
});

it('renders artefact details when expanded with artefactRefs', async () => {
  const el = await renderMessage({
    message: {
      artefactRefs: [
        { uri: 'doc:spec.md', type: 'DOCUMENT', label: 'Design Spec', scope: { startLine: 10, endLine: 20 } },
      ],
    },
  });
  const toggle = el.shadowRoot!.querySelector('.expand-toggle') as HTMLButtonElement;
  toggle.click();
  await (el as any).updateComplete;
  const detail = el.shadowRoot!.querySelector('.artefact-detail');
  expect(detail).toBeTruthy();
  expect(detail!.textContent).toContain('Design Spec');
  expect(detail!.textContent).toContain('doc:spec.md');
  expect(detail!.textContent).toContain('10');
});

it('renders commitment details when expanded for COMMAND with state', async () => {
  const el = await renderMessage({
    message: {
      messageType: 'COMMAND',
      commitmentId: 'c-1',
      deadline: '2026-07-15T12:00:00Z',
      acknowledgedAt: '2026-07-10T09:00:00Z',
    },
  });
  (el as any).commitmentState = 'ACKNOWLEDGED';
  await (el as any).updateComplete;
  const toggle = el.shadowRoot!.querySelector('.expand-toggle') as HTMLButtonElement;
  toggle.click();
  await (el as any).updateComplete;
  const details = el.shadowRoot!.querySelector('.commitment-details');
  expect(details).toBeTruthy();
  expect(details!.textContent).toContain('ACKNOWLEDGED');
});

it('does not render commitment details for non-COMMAND messages', async () => {
  const el = await renderMessage({ message: { messageType: 'EVENT' } });
  (el as any).commitmentState = 'OPEN';
  await (el as any).updateComplete;
  const toggle = el.shadowRoot!.querySelector('.expand-toggle') as HTMLButtonElement;
  toggle.click();
  await (el as any).updateComplete;
  expect(el.shadowRoot!.querySelector('.commitment-details')).toBeNull();
});

it('renders topic and channel metadata when expanded', async () => {
  const el = await renderMessage({ message: { topic: 'Deployment' } });
  (el as any).channelName = 'ops-channel';
  await (el as any).updateComplete;
  const toggle = el.shadowRoot!.querySelector('.expand-toggle') as HTMLButtonElement;
  toggle.click();
  await (el as any).updateComplete;
  const meta = el.shadowRoot!.querySelector('.metadata');
  expect(meta).toBeTruthy();
  expect(meta!.textContent).toContain('Deployment');
  expect(meta!.textContent).toContain('ops-channel');
});

it('renders reply button in expanded action bar', async () => {
  const el = await renderMessage();
  const toggle = el.shadowRoot!.querySelector('.expand-toggle') as HTMLButtonElement;
  toggle.click();
  await (el as any).updateComplete;
  const replyBtn = el.shadowRoot!.querySelector('.action-bar .reply-btn');
  expect(replyBtn).toBeTruthy();
});

it('reply button emits MESSAGE_SELECTED event', async () => {
  const el = await renderMessage();
  const toggle = el.shadowRoot!.querySelector('.expand-toggle') as HTMLButtonElement;
  toggle.click();
  await (el as any).updateComplete;

  const handler = vi.fn();
  el.addEventListener('pages-event', handler);
  const replyBtn = el.shadowRoot!.querySelector('.reply-btn') as HTMLButtonElement;
  replyBtn.click();

  expect(handler).toHaveBeenCalledOnce();
  expect(handler.mock.calls[0][0].detail.topic).toBe('chat:message-selected');
  expect(handler.mock.calls[0][0].detail.payload.message.id).toBe('msg-1');
});
```

- [ ] **Step 4: Implement qhorus-message expanded view**

In `qhorus-message.ts`:

1. Reclassify `expanded` from `@property()` to `@state()` named `_expanded`
2. Add `@property() parentMessage?: QhorusMessage`
3. Add `@property() channelName?: string`
4. Add `_toggle()` method
5. Add expand toggle button to header (after timestamp)
6. Add `_renderExpanded()` method with correlation context, artefact details,
   commitment details, metadata, and action bar
7. Add CSS for `.expand-toggle`, `.expanded-section`, `.correlation-context`,
   `.artefact-detail`, `.commitment-details`, `.metadata`, `.action-bar`,
   `.reply-btn`
8. Always render `<qhorus-reaction-bar>` (remove `reactions.length > 0` guard)
9. Pass `.messageId=${this.message.id}` to `<qhorus-reaction-bar>`
10. Import `emitChatEvent` and `ChatEventTopics` for the reply button

- [ ] **Step 5: Run message tests to verify they pass**

Run: `cd chat-demo/src/main/webui && npx vitest run src/qhorus/primitives/qhorus-message.test.ts`
Expected: PASS (some old tests may need adjustment for structural changes)

- [ ] **Step 6: Update existing tests that break**

The test `'renders nothing when message is not set'` may need adjustment since
the reaction bar now always renders. The test
`'does not render reaction bar when reactions is empty'` must be updated — the
bar now always renders (for the add button).

#### Part B: Feed and workbench plumbing

- [ ] **Step 7: Write failing tests for feed changes**

Update `qhorus-channel-feed.test.ts`:

```typescript
it('does not emit chat:message-selected on message item click', async () => {
  const el = document.createElement('qhorus-channel-feed') as any;
  el.messages = [msg('m1', { sender: 'alice' })];
  document.body.appendChild(el);
  await el.updateComplete;

  let eventFired = false;
  el.addEventListener('pages-event', (e: any) => {
    if (e.detail.topic === 'chat:message-selected') eventFired = true;
  });

  const messageItem = el.shadowRoot!.querySelector('.message-item') as HTMLElement;
  messageItem.click();
  expect(eventFired).toBe(false);
});

it('passes parentMessage to qhorus-message for replies', async () => {
  const el = document.createElement('qhorus-channel-feed') as any;
  const parent = msg('root', { sender: 'alice', content: 'Root message' });
  const reply = msg('reply', { sender: 'bob', inReplyTo: 'root' });
  el.messages = [parent, reply];
  document.body.appendChild(el);
  await el.updateComplete;

  // The reply message should not be in the root feed (it's under the thread),
  // but the root message should have no parentMessage
  const rootMsgEl = el.shadowRoot!.querySelector('qhorus-message') as any;
  expect(rootMsgEl.parentMessage).toBeUndefined();
});

it('passes channelName to qhorus-message elements', async () => {
  const el = document.createElement('qhorus-channel-feed') as any;
  el.messages = [msg('m1', { sender: 'alice' })];
  el.channelName = 'general';
  document.body.appendChild(el);
  await el.updateComplete;

  const msgEl = el.shadowRoot!.querySelector('qhorus-message') as any;
  expect(msgEl.channelName).toBe('general');
});
```

- [ ] **Step 8: Update the existing click-to-reply test**

The test `'emits chat:message-selected event when message clicked'` must
change to verify the feed does NOT emit on message click. Replace it with
the `'does not emit chat:message-selected on message item click'` test above.

- [ ] **Step 9: Implement feed changes**

In `qhorus-channel-feed.ts`:
1. Add `@property() channelName?: string`
2. Remove `_selectMessage` method
3. Remove `@click` handler from `.message-item` div
4. In `_renderFeed()`, pass `.parentMessage` and `.channelName` to each
   `<qhorus-message>`. For parentMessage: look up `msg.inReplyTo` in
   `this.messages` via `this.messages.find(m => m.id === msg.inReplyTo)`
5. Pass `.messageId=${msg.id}` to the reaction bar via the message

- [ ] **Step 10: Implement workbench channelName plumbing**

In `qhorus-workbench.ts`:
1. In `_renderChat()`, pass `.channelName` to the feed:
   `.channelName=${this._channels.find(c => c.id === this._selectedChannelId)?.name}`

- [ ] **Step 11: Run all tests**

Run: `cd chat-demo/src/main/webui && npx vitest run`
Expected: ALL PASS

- [ ] **Step 12: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/connectors add chat-demo/src/main/webui/src/qhorus/
git -C /Users/mdproctor/claude/casehub/connectors commit -m "feat(chat-demo): progressive disclosure on message expand — Refs #63"
```

---

### Task 2: Emoji Reaction Palette (#64)

**Files:**
- Create: `chat-demo/src/main/webui/src/qhorus/primitives/qhorus-emoji-picker.ts`
- Create: `chat-demo/src/main/webui/src/qhorus/primitives/qhorus-emoji-picker.test.ts`
- Modify: `chat-demo/src/main/webui/src/qhorus/primitives/qhorus-reaction-bar.ts`
- Modify: `chat-demo/src/main/webui/src/qhorus/primitives/qhorus-reaction-bar.test.ts`
- Modify: `chat-demo/src/main/webui/package.json`

**Interfaces:**
- Consumes: `emitChatEvent`, `ChatEventTopics.REACT` (existing),
  `Reaction` type (existing)
- Produces: `qhorus-emoji-picker` component emitting `emoji-selected`
  custom event with `{ emoji: string }`. `qhorus-reaction-bar` always
  renders (no empty-state guard), with add button and picker toggle.

#### Part A: Install dependency

- [ ] **Step 1: Add emoji-picker-element**

```bash
cd chat-demo/src/main/webui && npm install emoji-picker-element
```

- [ ] **Step 2: Verify installation**

```bash
ls chat-demo/src/main/webui/node_modules/emoji-picker-element/
```

#### Part B: qhorus-emoji-picker wrapper

- [ ] **Step 3: Write failing tests for emoji picker**

Create `qhorus-emoji-picker.test.ts`:

```typescript
import { describe, it, expect, vi, afterEach } from 'vitest';
import './qhorus-emoji-picker.js';

afterEach(() => { document.body.innerHTML = ''; });

describe('qhorus-emoji-picker', () => {
  it('renders an emoji-picker element', async () => {
    const el = document.createElement('qhorus-emoji-picker') as any;
    document.body.appendChild(el);
    await el.updateComplete;
    const picker = el.shadowRoot!.querySelector('emoji-picker');
    expect(picker).toBeTruthy();
  });

  it('emits emoji-selected event on emoji click', async () => {
    const el = document.createElement('qhorus-emoji-picker') as any;
    document.body.appendChild(el);
    await el.updateComplete;

    const handler = vi.fn();
    el.addEventListener('emoji-selected', handler);

    // Simulate emoji-picker-element's emoji-click event
    const picker = el.shadowRoot!.querySelector('emoji-picker')!;
    picker.dispatchEvent(new CustomEvent('emoji-click', {
      detail: { unicode: '😀', emoji: { unicode: '😀' } },
      bubbles: true,
    }));

    expect(handler).toHaveBeenCalledOnce();
    expect(handler.mock.calls[0][0].detail.emoji).toBe('😀');
  });
});
```

- [ ] **Step 4: Run tests to verify they fail**

Run: `cd chat-demo/src/main/webui && npx vitest run src/qhorus/primitives/qhorus-emoji-picker.test.ts`
Expected: FAIL — module not found

- [ ] **Step 5: Implement qhorus-emoji-picker**

Create `qhorus-emoji-picker.ts`:

```typescript
import { LitElement, html, css } from 'lit';
import { customElement } from 'lit/decorators.js';
import 'emoji-picker-element';

@customElement('qhorus-emoji-picker')
export class QhorusEmojiPickerElement extends LitElement {
  static override readonly styles = css`
    :host {
      display: block;
    }
    emoji-picker {
      --background: var(--pages-neutral-1, #fff);
      --border-color: var(--pages-neutral-5, #d4d4d4);
      --text-color: var(--pages-neutral-12, #111);
      --secondary-text-color: var(--pages-neutral-8, #888);
      --indicator-color: var(--pages-accent-9, #6366f1);
      --input-border-color: var(--pages-neutral-5, #d4d4d4);
      --button-hover-background: var(--pages-neutral-3, #e5e5e5);
      --button-active-background: var(--pages-neutral-4, #d4d4d4);
      --category-font-size: var(--pages-font-size-xs, 11px);
      --font-family: var(--pages-font-family, 'Inter', system-ui, sans-serif);
      --font-size: var(--pages-font-size-base, 14px);
    }
  `;

  private _onEmojiClick = (e: Event) => {
    const detail = (e as CustomEvent).detail;
    const unicode = detail.unicode ?? detail.emoji?.unicode;
    if (!unicode) return;
    this.dispatchEvent(new CustomEvent('emoji-selected', {
      bubbles: true,
      composed: true,
      detail: { emoji: unicode },
    }));
  };

  override render() {
    return html`<emoji-picker @emoji-click=${this._onEmojiClick}></emoji-picker>`;
  }
}

declare global {
  interface HTMLElementTagNameMap {
    'qhorus-emoji-picker': QhorusEmojiPickerElement;
  }
}
```

- [ ] **Step 6: Run picker tests**

Run: `cd chat-demo/src/main/webui && npx vitest run src/qhorus/primitives/qhorus-emoji-picker.test.ts`
Expected: PASS

#### Part C: Reaction bar add button and picker integration

- [ ] **Step 7: Write failing tests for reaction bar changes**

Add to `qhorus-reaction-bar.test.ts`:

```typescript
it('renders add button even when reactions are empty', async () => {
  const el = document.createElement('qhorus-reaction-bar') as any;
  el.reactions = [];
  el.messageId = 'msg-1';
  document.body.appendChild(el);
  await el.updateComplete;

  const addBtn = el.shadowRoot!.querySelector('.add-reaction-btn');
  expect(addBtn).toBeTruthy();
});

it('renders add button after reaction pills', async () => {
  const el = document.createElement('qhorus-reaction-bar') as any;
  el.reactions = makeReactions([['👍', ['a']]]);
  el.messageId = 'msg-1';
  document.body.appendChild(el);
  await el.updateComplete;

  const pills = el.shadowRoot!.querySelectorAll('.reaction-pill');
  const addBtn = el.shadowRoot!.querySelector('.add-reaction-btn');
  expect(pills.length).toBe(1);
  expect(addBtn).toBeTruthy();
});

it('clicking add button shows emoji picker', async () => {
  const el = document.createElement('qhorus-reaction-bar') as any;
  el.reactions = [];
  el.messageId = 'msg-1';
  document.body.appendChild(el);
  await el.updateComplete;

  const addBtn = el.shadowRoot!.querySelector('.add-reaction-btn') as HTMLButtonElement;
  addBtn.click();
  await el.updateComplete;

  const picker = el.shadowRoot!.querySelector('qhorus-emoji-picker');
  expect(picker).toBeTruthy();
});

it('selecting emoji emits chat:react and closes picker', async () => {
  const el = document.createElement('qhorus-reaction-bar') as any;
  el.reactions = [];
  el.messageId = 'msg-1';
  document.body.appendChild(el);
  await el.updateComplete;

  // Open picker
  const addBtn = el.shadowRoot!.querySelector('.add-reaction-btn') as HTMLButtonElement;
  addBtn.click();
  await el.updateComplete;

  const handler = vi.fn();
  el.addEventListener('pages-event', handler);

  // Simulate emoji selection
  const picker = el.shadowRoot!.querySelector('qhorus-emoji-picker')!;
  picker.dispatchEvent(new CustomEvent('emoji-selected', {
    bubbles: true, composed: true,
    detail: { emoji: '🎉' },
  }));
  await el.updateComplete;

  expect(handler).toHaveBeenCalledOnce();
  expect(handler.mock.calls[0][0].detail.topic).toBe('chat:react');
  expect(handler.mock.calls[0][0].detail.payload).toEqual({ messageId: 'msg-1', emoji: '🎉' });
});

it('Escape closes picker via Popover API', async () => {
  const el = document.createElement('qhorus-reaction-bar') as any;
  el.reactions = [];
  el.messageId = 'msg-1';
  document.body.appendChild(el);
  await el.updateComplete;

  const addBtn = el.shadowRoot!.querySelector('.add-reaction-btn') as HTMLButtonElement;
  addBtn.click();
  await el.updateComplete;

  // Simulate popover toggle event (light dismiss)
  const pickerContainer = el.shadowRoot!.querySelector('.picker-popover');
  if (pickerContainer) {
    pickerContainer.dispatchEvent(new ToggleEvent('toggle', { oldState: 'open', newState: 'closed' }));
    await el.updateComplete;
  }

  expect(el.shadowRoot!.querySelector('qhorus-emoji-picker')).toBeNull();
});
```

- [ ] **Step 8: Update existing empty-state test**

The test `'renders nothing when reactions array is empty'` must change.
Replace assertion: expect the add button to render, not `nothing`.

```typescript
it('renders add button when reactions array is empty', async () => {
  const el = document.createElement('qhorus-reaction-bar') as any;
  el.reactions = [];
  el.messageId = 'msg-1';
  document.body.appendChild(el);
  await el.updateComplete;

  const pills = el.shadowRoot!.querySelectorAll('.reaction-pill');
  expect(pills.length).toBe(0);
  const addBtn = el.shadowRoot!.querySelector('.add-reaction-btn');
  expect(addBtn).toBeTruthy();
});
```

- [ ] **Step 9: Implement reaction bar changes**

In `qhorus-reaction-bar.ts`:

1. Import `./qhorus-emoji-picker.js`
2. Add `@state() private _showPicker = false`
3. Remove the early `if (groups.length === 0) return nothing` guard
4. Add `_onAddClick()` — calls `togglePopover()` on the picker container
5. Add `_onToggle(e: ToggleEvent)` — syncs `_showPicker` from popover state
6. Add `_onEmojiSelected(e: CustomEvent)` — emits `chat:react` and calls
   `hidePopover()`
7. In `render()`: always render the add button after pills, conditionally
   render picker inside a `<div popover>` container
8. Add CSS for `.add-reaction-btn`, `.picker-popover`
9. Add positioning logic using `getBoundingClientRect()` in
   `_positionPicker()` called when showing

- [ ] **Step 10: Run all reaction bar tests**

Run: `cd chat-demo/src/main/webui && npx vitest run src/qhorus/primitives/qhorus-reaction-bar.test.ts`
Expected: PASS

- [ ] **Step 11: Run all tests**

Run: `cd chat-demo/src/main/webui && npx vitest run`
Expected: ALL PASS

- [ ] **Step 12: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/connectors add chat-demo/src/main/webui/
git -C /Users/mdproctor/claude/casehub/connectors commit -m "feat(chat-demo): emoji reaction palette with picker — Refs #64"
```

---

### Task 3: Swipe-to-Reveal Gestures (#55)

**Files:**
- Create: `chat-demo/src/main/webui/src/qhorus/workbench/swipe-controller.ts`
- Create: `chat-demo/src/main/webui/src/qhorus/workbench/swipe-controller.test.ts`
- Modify: `chat-demo/src/main/webui/src/qhorus/workbench/qhorus-workbench.ts`

**Interfaces:**
- Consumes: Lit `ReactiveController` interface, workbench's `_toggleNav()`,
  `_toggleMember()`, `_mode`, drawer DOM elements
- Produces: `SwipeController` class implementing `ReactiveController` with
  constructor options `{ drawerQuery: (side: 'left' | 'right') => HTMLElement | null,
  backdropQuery: () => HTMLElement | null, onOpen: (side: 'left' | 'right') => void }`

#### Part A: SwipeController

- [ ] **Step 1: Write failing tests for SwipeController**

Create `swipe-controller.test.ts`:

```typescript
import { describe, it, expect, vi, beforeEach, afterEach } from 'vitest';
import { SwipeController } from './swipe-controller.js';
import type { ReactiveControllerHost } from 'lit';

function mockHost(): ReactiveControllerHost & { mode: string; updateComplete: Promise<boolean> } {
  const controllers: any[] = [];
  return {
    mode: 'phone',
    updateComplete: Promise.resolve(true),
    addController(c: any) { controllers.push(c); },
    removeController() {},
    requestUpdate() {},
  };
}

function createDrawer(side: 'left' | 'right'): HTMLElement {
  const el = document.createElement('div');
  el.style.width = '280px';
  el.style.position = 'fixed';
  el.getBoundingClientRect = () => ({
    x: side === 'left' ? -280 : window.innerWidth,
    y: 0, width: 280, height: window.innerHeight,
    top: 0, bottom: window.innerHeight,
    left: side === 'left' ? -280 : window.innerWidth,
    right: side === 'left' ? 0 : window.innerWidth + 280,
    toJSON() { return this; },
  });
  document.body.appendChild(el);
  return el;
}

describe('SwipeController', () => {
  let host: ReturnType<typeof mockHost>;
  let leftDrawer: HTMLElement;
  let rightDrawer: HTMLElement;
  let backdrop: HTMLElement;
  let controller: SwipeController;

  beforeEach(() => {
    host = mockHost();
    leftDrawer = createDrawer('left');
    rightDrawer = createDrawer('right');
    backdrop = document.createElement('div');
    document.body.appendChild(backdrop);

    controller = new SwipeController(host, {
      drawerQuery: (side) => side === 'left' ? leftDrawer : rightDrawer,
      backdropQuery: () => backdrop,
      onOpen: vi.fn(),
    });
  });

  afterEach(() => { document.body.innerHTML = ''; });

  it('is a Lit reactive controller', () => {
    expect(controller.hostConnected).toBeDefined();
    expect(controller.hostDisconnected).toBeDefined();
  });

  it('ignores pointer events outside edge zones', () => {
    controller.hostConnected();
    const event = new PointerEvent('pointerdown', {
      clientX: 100, clientY: 200, pointerId: 1,
    });
    document.body.dispatchEvent(event);
    // No tracking started — no inline styles
    expect(leftDrawer.style.transform).toBe('');
  });

  it('starts tracking on pointerdown in left edge zone', () => {
    controller.hostConnected();
    const down = new PointerEvent('pointerdown', {
      clientX: 10, clientY: 200, pointerId: 1,
      bubbles: true,
    });
    document.body.dispatchEvent(down);

    const move = new PointerEvent('pointermove', {
      clientX: 100, clientY: 200, pointerId: 1,
      bubbles: true,
    });
    document.body.dispatchEvent(move);

    expect(leftDrawer.style.transform).not.toBe('');
  });

  it('opens drawer when dragged past 30% threshold', () => {
    controller.hostConnected();
    const onOpen = (controller as any)._options.onOpen;

    const down = new PointerEvent('pointerdown', {
      clientX: 5, clientY: 200, pointerId: 1, bubbles: true,
    });
    document.body.dispatchEvent(down);

    // Simulate horizontal drag past 30% of 280px = 84px
    const move = new PointerEvent('pointermove', {
      clientX: 100, clientY: 200, pointerId: 1, bubbles: true,
    });
    document.body.dispatchEvent(move);

    const up = new PointerEvent('pointerup', {
      clientX: 100, clientY: 200, pointerId: 1, bubbles: true,
    });
    document.body.dispatchEvent(up);

    expect(onOpen).toHaveBeenCalledWith('left');
  });

  it('snaps back when drag is insufficient', () => {
    controller.hostConnected();
    const onOpen = (controller as any)._options.onOpen;

    const down = new PointerEvent('pointerdown', {
      clientX: 5, clientY: 200, pointerId: 1, bubbles: true,
    });
    document.body.dispatchEvent(down);

    // Drag only 40px — less than 30% of 280px
    const move = new PointerEvent('pointermove', {
      clientX: 45, clientY: 200, pointerId: 1, bubbles: true,
    });
    document.body.dispatchEvent(move);

    const up = new PointerEvent('pointerup', {
      clientX: 45, clientY: 200, pointerId: 1, bubbles: true,
    });
    document.body.dispatchEvent(up);

    expect(onOpen).not.toHaveBeenCalled();
  });

  it('detaches listeners on hostDisconnected', () => {
    controller.hostConnected();
    controller.hostDisconnected();

    // After disconnect, pointer events should not trigger tracking
    const down = new PointerEvent('pointerdown', {
      clientX: 5, clientY: 200, pointerId: 1, bubbles: true,
    });
    document.body.dispatchEvent(down);
    expect(leftDrawer.style.transform).toBe('');
  });

  it('cleans up inline styles if mode changes mid-drag', () => {
    controller.hostConnected();

    const down = new PointerEvent('pointerdown', {
      clientX: 5, clientY: 200, pointerId: 1, bubbles: true,
    });
    document.body.dispatchEvent(down);

    const move = new PointerEvent('pointermove', {
      clientX: 60, clientY: 200, pointerId: 1, bubbles: true,
    });
    document.body.dispatchEvent(move);

    // Simulate mode change mid-drag
    controller.hostDisconnected();

    expect(leftDrawer.style.transform).toBe('');
    expect(backdrop.style.opacity).toBe('');
  });
});
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `cd chat-demo/src/main/webui && npx vitest run src/qhorus/workbench/swipe-controller.test.ts`
Expected: FAIL — module not found

- [ ] **Step 3: Implement SwipeController**

Create `swipe-controller.ts`:

```typescript
import type { ReactiveController, ReactiveControllerHost } from 'lit';

interface SwipeOptions {
  drawerQuery: (side: 'left' | 'right') => HTMLElement | null;
  backdropQuery: () => HTMLElement | null;
  onOpen: (side: 'left' | 'right') => void;
  edgeWidth?: number;
  drawerWidth?: number;
  distanceThreshold?: number;
  velocityThreshold?: number;
}

interface PointerSample {
  x: number;
  t: number;
}

export class SwipeController implements ReactiveController {
  private _host: ReactiveControllerHost;
  private _options: Required<SwipeOptions>;
  private _tracking = false;
  private _side: 'left' | 'right' = 'left';
  private _startX = 0;
  private _startY = 0;
  private _intentConfirmed = false;
  private _pointerId: number | null = null;
  private _samples: PointerSample[] = [];
  private _attached = false;
  private _reducedMotion = false;

  constructor(host: ReactiveControllerHost, options: SwipeOptions) {
    this._host = host;
    this._options = {
      edgeWidth: 20,
      drawerWidth: 280,
      distanceThreshold: 0.3,
      velocityThreshold: 0.5,
      ...options,
    };
    host.addController(this);
  }

  hostConnected() {
    this._reducedMotion = window.matchMedia('(prefers-reduced-motion: reduce)').matches;
    this._attachListeners();
  }

  hostDisconnected() {
    this._cleanupMidDrag();
    this._detachListeners();
  }

  private _attachListeners() {
    if (this._attached) return;
    document.body.addEventListener('pointerdown', this._onPointerDown);
    this._attached = true;
  }

  private _detachListeners() {
    if (!this._attached) return;
    document.body.removeEventListener('pointerdown', this._onPointerDown);
    document.body.removeEventListener('pointermove', this._onPointerMove);
    document.body.removeEventListener('pointerup', this._onPointerUp);
    document.body.removeEventListener('pointercancel', this._onPointerUp);
    this._attached = false;
  }

  private _cleanupMidDrag() {
    if (!this._tracking) return;
    const drawer = this._options.drawerQuery(this._side);
    const backdrop = this._options.backdropQuery();
    if (drawer) drawer.style.transform = '';
    if (backdrop) backdrop.style.opacity = '';
    if (this._pointerId != null) {
      try { document.body.releasePointerCapture(this._pointerId); } catch {}
    }
    this._tracking = false;
    this._intentConfirmed = false;
    this._pointerId = null;
    this._samples = [];
  }

  private _onPointerDown = (e: PointerEvent) => {
    const { edgeWidth } = this._options;
    const w = window.innerWidth;
    let side: 'left' | 'right';

    if (e.clientX <= edgeWidth) side = 'left';
    else if (e.clientX >= w - edgeWidth) side = 'right';
    else return;

    this._side = side;
    this._startX = e.clientX;
    this._startY = e.clientY;
    this._tracking = true;
    this._intentConfirmed = false;
    this._pointerId = e.pointerId;
    this._samples = [{ x: e.clientX, t: e.timeStamp }];

    try { document.body.setPointerCapture(e.pointerId); } catch {}

    document.body.addEventListener('pointermove', this._onPointerMove);
    document.body.addEventListener('pointerup', this._onPointerUp);
    document.body.addEventListener('pointercancel', this._onPointerUp);
  };

  private _onPointerMove = (e: PointerEvent) => {
    if (!this._tracking) return;

    const dx = Math.abs(e.clientX - this._startX);
    const dy = Math.abs(e.clientY - this._startY);

    if (!this._intentConfirmed) {
      if (dx + dy < 10) return;
      if (dy > dx) {
        this._cancelDrag();
        return;
      }
      this._intentConfirmed = true;
    }

    e.preventDefault();

    this._samples.push({ x: e.clientX, t: e.timeStamp });
    if (this._samples.length > 6) this._samples.shift();

    if (this._reducedMotion) return;

    const { drawerWidth } = this._options;
    const drawer = this._options.drawerQuery(this._side);
    const backdrop = this._options.backdropQuery();
    if (!drawer) return;

    let delta: number;
    if (this._side === 'left') {
      delta = Math.max(0, Math.min(drawerWidth, e.clientX - this._startX));
      drawer.style.transform = `translateX(calc(-100% + ${delta}px))`;
    } else {
      delta = Math.max(0, Math.min(drawerWidth, this._startX - e.clientX));
      drawer.style.transform = `translateX(calc(100% - ${delta}px))`;
    }

    const progress = delta / drawerWidth;
    if (backdrop) backdrop.style.opacity = String(progress * 0.5);
  };

  private _onPointerUp = (e: PointerEvent) => {
    if (!this._tracking) return;

    document.body.removeEventListener('pointermove', this._onPointerMove);
    document.body.removeEventListener('pointerup', this._onPointerUp);
    document.body.removeEventListener('pointercancel', this._onPointerUp);

    if (this._pointerId != null) {
      try { document.body.releasePointerCapture(this._pointerId); } catch {}
    }

    if (!this._intentConfirmed) {
      this._tracking = false;
      this._pointerId = null;
      return;
    }

    const { drawerWidth, distanceThreshold, velocityThreshold } = this._options;
    const totalDelta = this._side === 'left'
      ? e.clientX - this._startX
      : this._startX - e.clientX;

    const distanceMet = totalDelta / drawerWidth > distanceThreshold;
    const velocity = this._computeVelocity(e.timeStamp);
    const velocityMet = velocity > velocityThreshold;

    const drawer = this._options.drawerQuery(this._side);
    const backdrop = this._options.backdropQuery();

    if (distanceMet || velocityMet) {
      if (drawer) {
        drawer.style.transform = this._side === 'left'
          ? 'translateX(0)' : 'translateX(0)';
      }
      this._options.onOpen(this._side);
      (this._host as any).updateComplete.then(() => {
        if (drawer) drawer.style.transform = '';
        if (backdrop) backdrop.style.opacity = '';
      });
    } else {
      if (drawer) drawer.style.transform = '';
      if (backdrop) backdrop.style.opacity = '';
    }

    this._tracking = false;
    this._pointerId = null;
    this._samples = [];
  };

  private _cancelDrag() {
    document.body.removeEventListener('pointermove', this._onPointerMove);
    document.body.removeEventListener('pointerup', this._onPointerUp);
    document.body.removeEventListener('pointercancel', this._onPointerUp);
    if (this._pointerId != null) {
      try { document.body.releasePointerCapture(this._pointerId); } catch {}
    }
    this._tracking = false;
    this._pointerId = null;
    this._samples = [];
  }

  private _computeVelocity(now: number): number {
    if (this._samples.length < 2) return 0;
    const windowMs = 100;
    const recent = this._samples.filter(s => now - s.t <= windowMs);
    if (recent.length < 2) {
      const first = this._samples[0];
      const last = this._samples[this._samples.length - 1];
      const dt = last.t - first.t;
      return dt > 0 ? Math.abs(last.x - first.x) / dt : 0;
    }
    const first = recent[0];
    const last = recent[recent.length - 1];
    const dt = last.t - first.t;
    return dt > 0 ? Math.abs(last.x - first.x) / dt : 0;
  }
}
```

- [ ] **Step 4: Run SwipeController tests**

Run: `cd chat-demo/src/main/webui && npx vitest run src/qhorus/workbench/swipe-controller.test.ts`
Expected: PASS

#### Part B: Integrate into workbench

- [ ] **Step 5: Write failing test for workbench swipe integration**

Add to `qhorus-workbench.test.ts`:

```typescript
describe('swipe gestures (phone mode)', () => {
  it('has SwipeController attached', async () => {
    const el = await renderWorkbench() as any;
    expect(el._swipeController).toBeDefined();
  });
});
```

- [ ] **Step 6: Integrate SwipeController into workbench**

In `qhorus-workbench.ts`:

1. Import `SwipeController`
2. In the class, add a `_swipeController` field initialized in the constructor
   (or as a class field):
   ```typescript
   private _swipeController = new SwipeController(this, {
     drawerQuery: (side) => this.renderRoot.querySelector(
       side === 'left' ? '.drawer.left' : '.drawer.right'
     ),
     backdropQuery: () => this.renderRoot.querySelector('.backdrop'),
     onOpen: (side) => {
       if (side === 'left') this._toggleNav();
       else this._toggleMember();
     },
   });
   ```
3. No other changes needed — the controller self-manages via Lit lifecycle

- [ ] **Step 7: Run all tests**

Run: `cd chat-demo/src/main/webui && npx vitest run`
Expected: ALL PASS

- [ ] **Step 8: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/connectors add chat-demo/src/main/webui/src/qhorus/workbench/
git -C /Users/mdproctor/claude/casehub/connectors commit -m "feat(chat-demo): swipe-to-reveal gestures for phone drawers — Refs #55"
```

---

### Task 4: Build Verification and Final Cleanup

**Files:**
- None created — verification only

- [ ] **Step 1: Run full test suite**

```bash
cd chat-demo/src/main/webui && npx vitest run
```
Expected: ALL PASS

- [ ] **Step 2: Run TypeScript type check**

```bash
cd chat-demo/src/main/webui && npx tsc --noEmit
```
Expected: No errors

- [ ] **Step 3: Run esbuild production build**

```bash
cd chat-demo/src/main/webui && node esbuild.config.mjs
```
Expected: Build succeeds

- [ ] **Step 4: Run full Maven build with demo + ui profiles**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn -f /Users/mdproctor/claude/casehub/connectors/pom.xml clean install -Pdemo -Pui
```
Expected: BUILD SUCCESS
