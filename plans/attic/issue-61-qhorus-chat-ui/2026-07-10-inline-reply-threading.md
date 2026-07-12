# Inline Reply Threading Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> executing-plans to implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural editing.
> Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #79 — feat(chat-demo): replace Flat/Threaded toggle with inline reply threading
**Goal:** Replace the broken Flat/Threaded toggle with inline reply threading that uses `inReplyTo` data

**Architecture:** Remove the toggle and threaded-mode code from channel-feed. Add a composable `_separateRootsAndReplies()` method that partitions messages into roots and a replies-by-parent map. Render roots with sender grouping (existing logic), and after each root that has replies, render an inline `<qhorus-thread>`. Compute `replyCount` in the adapter from `inReplyTo` relationships.

**Tech Stack:** Lit 3, TypeScript, Vitest + happy-dom

## Global Constraints

- All CSS variables must use `--pages-neutral-*`, `--pages-accent-*` etc. (enforced by theme-variables.test.ts)
- `<qhorus-thread>` primitive is unchanged — reuse as-is
- The `_separateRootsAndReplies()` method must be a standalone composable step (not baked into render) so Topics mode (qhorus#328) can layer a filter before it

---

### Task 1: Adapter — compute replyCount from inReplyTo

**Files:**
- Modify: `chat-demo/src/main/webui/src/qhorus/workbench/chat-demo-adapter.ts`
- Test: `chat-demo/src/main/webui/src/qhorus/workbench/chat-demo-adapter.test.ts`

**Interfaces:**
- Produces: `replyCount` field on each `QhorusMessage` is populated after snapshot/append

- [ ] **Step 1: Write failing tests**

Add to `chat-demo-adapter.test.ts`:

```typescript
it('computes replyCount from inReplyTo references', () => {
  const adapter = new ChatDemoAdapter();
  adapter.applyOp({
    op: 'snapshot', dataset: 'messages',
    rows: [
      ['ch-1', 'msg-1', null, 'alice', 'Root message', '2026-07-07T12:00:00Z'],
      ['ch-1', 'msg-2', 'msg-1', 'bob', 'Reply 1', '2026-07-07T12:01:00Z'],
      ['ch-1', 'msg-3', 'msg-1', 'carol', 'Reply 2', '2026-07-07T12:02:00Z'],
      ['ch-1', 'msg-4', null, 'dave', 'Standalone', '2026-07-07T12:03:00Z'],
    ],
  });
  expect(adapter.messages.find(m => m.id === 'msg-1')!.replyCount).toBe(2);
  expect(adapter.messages.find(m => m.id === 'msg-4')!.replyCount).toBe(0);
  expect(adapter.messages.find(m => m.id === 'msg-2')!.replyCount).toBe(0);
});

it('updates replyCount after append adds a reply', () => {
  const adapter = new ChatDemoAdapter();
  adapter.applyOp({
    op: 'snapshot', dataset: 'messages',
    rows: [['ch-1', 'msg-1', null, 'alice', 'Root', '2026-07-07T12:00:00Z']],
  });
  expect(adapter.messages[0].replyCount).toBe(0);
  adapter.applyOp({
    op: 'append', dataset: 'messages',
    rows: [['ch-1', 'msg-2', 'msg-1', 'bob', 'Reply', '2026-07-07T12:01:00Z']],
  });
  expect(adapter.messages.find(m => m.id === 'msg-1')!.replyCount).toBe(1);
});
```

- [ ] **Step 2: Run tests — expect FAIL**

Run: `npx vitest run src/qhorus/workbench/chat-demo-adapter.test.ts`
Expected: 2 failures — replyCount is always 0

- [ ] **Step 3: Implement replyCount computation**

Add a `_recomputeReplyCounts()` method to `ChatDemoAdapter` and call it at the end of `_applyMessages`:

```typescript
private _recomputeReplyCounts() {
  const counts = new Map<string, number>();
  for (const m of this.messages) {
    if (m.inReplyTo) {
      counts.set(m.inReplyTo, (counts.get(m.inReplyTo) ?? 0) + 1);
    }
  }
  this.messages = this.messages.map(m => ({
    ...m,
    replyCount: counts.get(m.id) ?? 0,
  }));
}
```

Call `this._recomputeReplyCounts()` at the end of `_applyMessages()`, after snapshot/append/remove.

- [ ] **Step 4: Run tests — expect PASS**

Run: `npx vitest run src/qhorus/workbench/chat-demo-adapter.test.ts`
Expected: All tests pass

- [ ] **Step 5: Commit**

```bash
git add chat-demo/src/main/webui/src/qhorus/workbench/chat-demo-adapter.ts chat-demo/src/main/webui/src/qhorus/workbench/chat-demo-adapter.test.ts
git commit -m "feat(chat-demo): compute replyCount from inReplyTo — Refs #79"
```

---

### Task 2: Channel-feed — remove toggle, add inline reply threading

**Files:**
- Modify: `chat-demo/src/main/webui/src/qhorus/composites/qhorus-channel-feed.ts`
- Test: `chat-demo/src/main/webui/src/qhorus/composites/qhorus-channel-feed.test.ts`

**Interfaces:**
- Consumes: `QhorusMessage.inReplyTo` and `QhorusMessage.replyCount` (from Task 1)
- Produces: renders `<qhorus-thread>` inline after root messages that have replies

- [ ] **Step 1: Write failing tests for inline reply threading**

Replace the test file contents. Remove all tests that reference `mode`, `mode-toggle`, `threaded`, or `correlationId`. Add new tests:

```typescript
it('separates root messages from replies', async () => {
  const el = document.createElement('qhorus-channel-feed') as any;
  el.messages = [
    msg('root', { sender: 'alice' }),
    msg('reply1', { sender: 'bob', inReplyTo: 'root' }),
    msg('reply2', { sender: 'carol', inReplyTo: 'root' }),
    msg('standalone', { sender: 'dave' }),
  ];
  document.body.appendChild(el);
  await el.updateComplete;

  // Root messages rendered, replies not rendered as standalone
  const groups = el.shadowRoot!.querySelectorAll('.message-group');
  const allMessages = el.shadowRoot!.querySelectorAll('qhorus-message');
  // root + standalone rendered in groups; replies rendered inside thread
  const threads = el.shadowRoot!.querySelectorAll('qhorus-thread');
  expect(threads.length).toBe(1);
});

it('renders qhorus-thread after root message with replies', async () => {
  const el = document.createElement('qhorus-channel-feed') as any;
  el.messages = [
    msg('m1', { sender: 'alice', replyCount: 2 }),
    msg('r1', { sender: 'bob', inReplyTo: 'm1' }),
    msg('r2', { sender: 'carol', inReplyTo: 'm1' }),
  ];
  document.body.appendChild(el);
  await el.updateComplete;

  const thread = el.shadowRoot!.querySelector('qhorus-thread') as any;
  expect(thread).toBeTruthy();
  expect(thread.rootMessage.id).toBe('m1');
  expect(thread.replies.length).toBe(2);
});

it('does not render toggle buttons', async () => {
  const el = document.createElement('qhorus-channel-feed') as any;
  el.messages = [msg('1')];
  document.body.appendChild(el);
  await el.updateComplete;

  const toggle = el.shadowRoot!.querySelector('.mode-toggle');
  expect(toggle).toBeNull();
});

it('renders messages without replies as normal groups', async () => {
  const el = document.createElement('qhorus-channel-feed') as any;
  el.messages = [
    msg('1', { sender: 'alice', createdAt: '2026-07-07T12:00:00Z' }),
    msg('2', { sender: 'alice', createdAt: '2026-07-07T12:00:30Z' }),
    msg('3', { sender: 'bob', createdAt: '2026-07-07T12:01:00Z' }),
  ];
  document.body.appendChild(el);
  await el.updateComplete;

  const groups = el.shadowRoot!.querySelectorAll('.message-group');
  expect(groups.length).toBe(2);
  const threads = el.shadowRoot!.querySelectorAll('qhorus-thread');
  expect(threads.length).toBe(0);
});
```

- [ ] **Step 2: Run tests — expect FAIL**

Run: `npx vitest run src/qhorus/composites/qhorus-channel-feed.test.ts`
Expected: Failures — toggle still exists, inline threading not implemented

- [ ] **Step 3: Implement changes to channel-feed**

**Remove:**
- `ThreadGroup` interface
- `mode` property
- `_setMode()` method
- `_groupThreaded()` method
- `_renderThreaded()` method
- `.toolbar`, `.mode-toggle` CSS rules
- Toggle UI from `render()`

**Add `_separateRootsAndReplies()` method** (composable step for future Topics mode):

```typescript
private _separateRootsAndReplies(): {
  roots: QhorusMessage[];
  repliesByParent: Map<string, QhorusMessage[]>;
} {
  const repliesByParent = new Map<string, QhorusMessage[]>();
  const roots: QhorusMessage[] = [];

  for (const m of this.messages) {
    if (m.inReplyTo) {
      const list = repliesByParent.get(m.inReplyTo) ?? [];
      list.push(m);
      repliesByParent.set(m.inReplyTo, list);
    } else {
      roots.push(m);
    }
  }
  return { roots, repliesByParent };
}
```

**Modify `_groupFlat()`** to accept a messages parameter instead of using `this.messages`:

```typescript
private _groupFlat(messages: QhorusMessage[]): MessageGroup[] {
  const groups: MessageGroup[] = [];
  const TWO_MINUTES = 2 * 60 * 1000;

  for (const msg of messages) {
    const last = groups[groups.length - 1];
    if (last && last.sender === msg.sender) {
      const lastTime = new Date(last.messages[last.messages.length - 1].createdAt).getTime();
      const thisTime = new Date(msg.createdAt).getTime();
      if (thisTime - lastTime < TWO_MINUTES) {
        last.messages = [...last.messages, msg];
        continue;
      }
    }
    groups.push({ sender: msg.sender, actorType: msg.actorType, messages: [msg] });
  }
  return groups;
}
```

**Update `render()` and `_renderFlat()`** to use inline threading:

```typescript
override render() {
  return html`
    <div class="feed">
      ${this.messages.length === 0 ? html`
        <div class="empty">No messages yet</div>
      ` : this._renderFeed()}
    </div>
  `;
}

private _renderFeed() {
  const { roots, repliesByParent } = this._separateRootsAndReplies();
  return this._groupFlat(roots).map(group => html`
    <div class="message-group">
      <div class="message-group-header">
        <span class="group-sender">${group.sender}</span>
      </div>
      ${group.messages.map(msg => html`
        <div class="message-item" @click=${() => this._selectMessage(msg)}>
          <qhorus-message .message=${msg}
                          .reactions=${this._reactionsFor(msg.id)}
                          .showActorBadge=${group.messages.indexOf(msg) === 0}>
          </qhorus-message>
        </div>
        ${repliesByParent.has(msg.id) ? html`
          <qhorus-thread .rootMessage=${msg}
                         .replies=${repliesByParent.get(msg.id)!}>
          </qhorus-thread>
        ` : nothing}
      `)}
    </div>
  `);
}
```

- [ ] **Step 4: Run tests — expect PASS**

Run: `npx vitest run src/qhorus/composites/qhorus-channel-feed.test.ts`
Expected: All tests pass

- [ ] **Step 5: Run full test suite**

Run: `npx vitest run`
Expected: All tests pass (workbench tests that reference `mode` may need updating — see Task 3)

- [ ] **Step 6: Commit**

```bash
git add chat-demo/src/main/webui/src/qhorus/composites/qhorus-channel-feed.ts chat-demo/src/main/webui/src/qhorus/composites/qhorus-channel-feed.test.ts
git commit -m "feat(chat-demo): inline reply threading — replaces broken Flat/Threaded toggle — Refs #79"
```

---

### Task 3: Fix dependent tests and verify in Playwright

**Files:**
- Modify: any test files that reference `mode`, `flat`, `threaded`, or `mode-toggle` on the channel-feed
- Verify: Playwright at desktop width

- [ ] **Step 1: Run full test suite, fix any remaining failures**

Run: `npx vitest run`
Fix any tests that reference the removed `mode` property or toggle buttons.

- [ ] **Step 2: Verify in Playwright**

Navigate to http://localhost:8090/src/index.html, select general channel. Verify:
- No Flat/Threaded toggle visible
- Messages render chronologically with sender grouping
- If any messages have replies (`inReplyTo`), a collapsed thread appears below the root
- Clicking the thread expander shows indented replies

- [ ] **Step 3: Test sending a reply**

Click a message to set reply-to context, type a reply, send it. Verify the reply appears as part of the thread under the parent message.

- [ ] **Step 4: Full build**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn -f connectors clean install -Pdemo -Pui`
Expected: BUILD SUCCESS

- [ ] **Step 5: Commit any remaining fixes**

```bash
git add -A
git commit -m "fix(chat-demo): update tests for inline reply threading — Refs #79"
```
