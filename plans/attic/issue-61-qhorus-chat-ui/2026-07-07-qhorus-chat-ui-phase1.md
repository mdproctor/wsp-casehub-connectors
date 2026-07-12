# Qhorus Chat UI — Phase 1 Foundation Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** TBD — file before starting implementation
**Issue group:** TBD

**Goal:** Port chat-demo webui from raw HTMLElement + Shadow DOM to
Lit + pages dataset pipeline + blocks-ui accessibility mixins, with
qhorus-native data model rendering speech acts, actor types, and
correlation chains.

**Architecture:** Component family in three layers (primitives →
composites → workbench). Primitives are pure LitElement with props.
Composites extend LitElement and receive data via `@property()` setters
fed by the workbench's dataset pipeline. Workbench owns the data
connection (WebSocket adapter) and assembles composites via pages
`split()`/`dockBar()` layout primitives.

**Tech Stack:** Lit 3.3, Vite, vitest + jsdom, `@casehubio/blocks-ui-core`
(a11y mixins), `@casehubio/pages-runtime` + `@casehubio/pages-ui`
(layout DSL), `@casehubio/pages-data` (dataset pipeline), `marked`
(markdown rendering), `DOMPurify` (sanitization).

## Global Constraints

- **Lit version:** 3.3, matching `@casehubio/blocks-ui-core`
- **TypeScript:** `experimentalDecorators: true`, `useDefineForClassFields: false`
- **CSS:** `--pages-*` design tokens only — no hardcoded colors, fonts, or spacing
- **Events:** `pages-event` CustomEvent with `{topic, payload}` detail, `bubbles: true`, `composed: true`
- **Accessibility:** KeyboardShortcutMixin, LiveRegionMixin, RovingTabindexMixin, FocusTrapMixin from `@casehubio/blocks-ui-core`
- **Collections:** Immutable replace, never mutate (Lit reactivity requirement)
- **Build:** Vite dev server + HMR; esbuild production bundle (Vite uses esbuild internally)
- **Test:** vitest + jsdom, co-located test files (`*.test.ts`)
- **Message rendering:** Markdown via `marked` + `DOMPurify` sanitization
- **No topic features in Phase 1** — topic selector, topic navigator bar, topic view mode all require qhorus#328
- **Old code untouched until Task 9 (swap)** — all new code in `src/qhorus/`

## File Structure

```
chat-demo/src/main/webui/
├── package.json                    ← MODIFY: add Lit, blocks-ui-core, pages-data, marked, DOMPurify
├── vite.config.ts                  ← CREATE: Vite dev + build config
├── vitest.config.ts                ← MODIFY: add path aliases for monorepo packages
├── tsconfig.json                   ← MODIFY: add experimentalDecorators, paths
├── src/
│   ├── auth.ts                     ← KEEP (shared, used by new workbench)
│   ├── auth.test.ts                ← KEEP
│   ├── index.ts                    ← OLD (untouched until swap)
│   ├── responsive.ts               ← OLD (delete at swap)
│   ├── responsive.test.ts          ← OLD (delete at swap)
│   ├── layout-fit.test.ts          ← OLD (delete at swap)
│   ├── test-helpers.ts             ← OLD (delete at swap)
│   ├── panels/                     ← OLD (delete at swap)
│   └── qhorus/                     ← NEW
│       ├── types.ts                ← Domain types: QhorusMessage, QhorusChannel, etc.
│       ├── types.test.ts           ← Type guard and factory tests
│       ├── events.ts               ← Event topics, emitter helper, payload types
│       ├── events.test.ts          ← Event emission tests
│       ├── markdown.ts             ← Markdown renderer (marked + DOMPurify)
│       ├── markdown.test.ts        ← Sanitization tests
│       ├── primitives/
│       │   ├── qhorus-message.ts
│       │   ├── qhorus-message.test.ts
│       │   ├── qhorus-reaction-bar.ts
│       │   ├── qhorus-reaction-bar.test.ts
│       │   ├── qhorus-thread.ts
│       │   ├── qhorus-thread.test.ts
│       │   ├── qhorus-message-input.ts
│       │   └── qhorus-message-input.test.ts
│       ├── composites/
│       │   ├── qhorus-channel-feed.ts
│       │   ├── qhorus-channel-feed.test.ts
│       │   ├── qhorus-channel-nav.ts
│       │   ├── qhorus-channel-nav.test.ts
│       │   ├── qhorus-member-panel.ts
│       │   └── qhorus-member-panel.test.ts
│       ├── workbench/
│       │   ├── qhorus-workbench.ts
│       │   ├── qhorus-workbench.test.ts
│       │   ├── chat-demo-adapter.ts
│       │   └── chat-demo-adapter.test.ts
│       └── index.ts                ← NEW entry point (used at swap)
```

---

### Task 1: Scaffold, Types, and Shared Utilities

**Files:**
- Modify: `chat-demo/src/main/webui/package.json`
- Modify: `chat-demo/src/main/webui/tsconfig.json`
- Modify: `chat-demo/src/main/webui/vitest.config.ts`
- Create: `chat-demo/src/main/webui/vite.config.ts`
- Create: `chat-demo/src/main/webui/src/qhorus/types.ts`
- Create: `chat-demo/src/main/webui/src/qhorus/types.test.ts`
- Create: `chat-demo/src/main/webui/src/qhorus/events.ts`
- Create: `chat-demo/src/main/webui/src/qhorus/events.test.ts`
- Create: `chat-demo/src/main/webui/src/qhorus/markdown.ts`
- Create: `chat-demo/src/main/webui/src/qhorus/markdown.test.ts`

**Interfaces:**
- Consumes: Nothing — this is the foundation
- Produces:
  - `QhorusMessage` — `{id, channelId, sender, messageType, actorType, content, correlationId?, inReplyTo?, replyCount, artefactRefs, target?, commitmentId?, deadline?, acknowledgedAt?, createdAt}`
  - `MessageType` — `'QUERY' | 'COMMAND' | 'RESPONSE' | 'STATUS' | 'DONE' | 'FAILURE' | 'DECLINE' | 'HANDOFF' | 'EVENT'`
  - `ActorType` — `'HUMAN' | 'AGENT' | 'SYSTEM'`
  - `CommitmentState` — `'OPEN' | 'ACKNOWLEDGED' | 'FULFILLED' | 'FAILED' | 'DECLINED' | 'DELEGATED' | 'EXPIRED'`
  - `QhorusChannel` — `{id, name, description?, semantic, allowedTypes?, deniedTypes?, paused}`
  - `ChannelSemantic` — `'APPEND' | 'COLLECT' | 'BARRIER' | 'EPHEMERAL' | 'LAST_WRITE'`
  - `Reaction` — `{messageId, emoji, actorId, createdAt}`
  - `ArtefactRef` — `{uri, type, label, scope?}`
  - `ChatEventTopics` — const object with all `chat:*` topic strings
  - `emitChatEvent(target, topic, payload)` — helper function
  - `renderMarkdown(content: string): string` — sanitized HTML string

- [ ] **Step 1: Install dependencies**

```bash
cd chat-demo/src/main/webui && npm install lit@^3.3 marked@^15 dompurify@^3 && npm install -D @types/dompurify@^3
```

Note: `@casehubio/blocks-ui-core`, `@casehubio/pages-runtime`, `@casehubio/pages-ui`,
`@casehubio/pages-data` are already available via `file:` links or will be added as
`file:` references matching the existing pattern in package.json:

```json
"@casehubio/blocks-ui-core": "file:../../../../../blocks-ui/packages/blocks-ui-core",
"@casehubio/pages-data": "file:../../../../../pages/packages/pages-data"
```

- [ ] **Step 2: Update tsconfig.json**

Add to `compilerOptions`:
```json
{
  "experimentalDecorators": true,
  "useDefineForClassFields": false
}
```

These are required for Lit decorators (`@customElement`, `@property`, `@state`).

- [ ] **Step 3: Create vite.config.ts**

```typescript
import { defineConfig } from 'vite';
import path from 'path';

const PAGES = path.resolve(__dirname, '../../../../../pages/packages');
const BLOCKS = path.resolve(__dirname, '../../../../../blocks-ui/packages');

export default defineConfig({
  resolve: {
    alias: {
      '@casehubio/blocks-ui-core': path.resolve(BLOCKS, 'blocks-ui-core/src'),
      '@casehubio/pages-ui-tokens': path.resolve(PAGES, 'pages-ui-tokens/src'),
      '@casehubio/pages-component': path.resolve(PAGES, 'pages-component/src'),
      '@casehubio/pages-data/dist/sse/sse-manager.js': path.resolve(PAGES, 'pages-data/src/sse/sse-manager.ts'),
      '@casehubio/pages-data': path.resolve(PAGES, 'pages-data/src'),
    },
  },
  esbuild: {
    target: 'es2022',
    tsconfigRaw: {
      compilerOptions: {
        experimentalDecorators: true,
        useDefineForClassFields: false,
      },
    },
  },
});
```

- [ ] **Step 4: Update vitest.config.ts**

Replace the existing vitest config with one that extends the Vite config:

```typescript
import { defineConfig, mergeConfig } from 'vitest/config';
import viteConfig from './vite.config';

export default mergeConfig(viteConfig, defineConfig({
  test: {
    environment: 'jsdom',
    globals: true,
    include: ['src/**/*.test.ts'],
  },
}));
```

- [ ] **Step 5: Write types.ts**

```typescript
// chat-demo/src/main/webui/src/qhorus/types.ts

export const MESSAGE_TYPES = [
  'QUERY', 'COMMAND', 'RESPONSE', 'STATUS', 'DONE',
  'FAILURE', 'DECLINE', 'HANDOFF', 'EVENT',
] as const;
export type MessageType = typeof MESSAGE_TYPES[number];

export const ACTOR_TYPES = ['HUMAN', 'AGENT', 'SYSTEM'] as const;
export type ActorType = typeof ACTOR_TYPES[number];

export const COMMITMENT_STATES = [
  'OPEN', 'ACKNOWLEDGED', 'FULFILLED', 'FAILED',
  'DECLINED', 'DELEGATED', 'EXPIRED',
] as const;
export type CommitmentState = typeof COMMITMENT_STATES[number];

export const CHANNEL_SEMANTICS = [
  'APPEND', 'COLLECT', 'BARRIER', 'EPHEMERAL', 'LAST_WRITE',
] as const;
export type ChannelSemantic = typeof CHANNEL_SEMANTICS[number];

export const ARTEFACT_TYPES = [
  'DOCUMENT', 'CODE', 'CASE', 'WORK_ITEM', 'CHANNEL',
  'DEBATE', 'MESSAGE', 'EXTERNAL',
] as const;
export type ArtefactType = typeof ARTEFACT_TYPES[number];

export interface SelectionScope {
  readonly startLine?: number;
  readonly endLine?: number;
  readonly startOffset?: number;
  readonly endOffset?: number;
  readonly selectedText?: string;
}

export interface ArtefactRef {
  readonly uri: string;
  readonly type: ArtefactType;
  readonly label: string;
  readonly scope?: SelectionScope;
}

export interface QhorusMessage {
  readonly id: string;
  readonly channelId: string;
  readonly sender: string;
  readonly messageType: MessageType;
  readonly actorType: ActorType;
  readonly content: string;
  readonly correlationId?: string;
  readonly inReplyTo?: string;
  readonly replyCount: number;
  readonly artefactRefs: readonly ArtefactRef[];
  readonly target?: string;
  readonly commitmentId?: string;
  readonly deadline?: string;
  readonly acknowledgedAt?: string;
  readonly createdAt: string;
}

export interface QhorusChannel {
  readonly id: string;
  readonly name: string;
  readonly description?: string;
  readonly semantic: ChannelSemantic;
  readonly allowedTypes?: readonly MessageType[];
  readonly deniedTypes?: readonly MessageType[];
  readonly paused: boolean;
}

export interface Reaction {
  readonly messageId: string;
  readonly emoji: string;
  readonly actorId: string;
  readonly createdAt: string;
}

export interface ChannelMember {
  readonly channelId: string;
  readonly memberId: string;
  readonly displayName: string;
  readonly role: 'PARTICIPANT' | 'OBSERVER' | 'MODERATOR';
}

export interface PresenceState {
  readonly memberId: string;
  readonly status: 'ONLINE' | 'AVAILABLE' | 'BUSY' | 'AWAY' | 'OFFLINE';
  readonly lastSeenAt?: string;
  readonly statusMessage?: string;
}

export function isTerminalMessageType(type: MessageType): boolean {
  return type === 'DONE' || type === 'FAILURE' || type === 'DECLINE' || type === 'HANDOFF';
}

export function isObligationCreating(type: MessageType): boolean {
  return type === 'COMMAND';
}

export function messageTypeCategory(type: MessageType): 'info' | 'obligation' | 'success' | 'danger' | 'warning' | 'transfer' | 'telemetry' {
  switch (type) {
    case 'QUERY': case 'RESPONSE': case 'STATUS': return 'info';
    case 'COMMAND': return 'obligation';
    case 'DONE': return 'success';
    case 'FAILURE': return 'danger';
    case 'DECLINE': return 'warning';
    case 'HANDOFF': return 'transfer';
    case 'EVENT': return 'telemetry';
  }
}

export function commitmentStateCategory(state: CommitmentState): 'active' | 'success' | 'danger' | 'neutral' | 'transfer' | 'warning' {
  switch (state) {
    case 'OPEN': return 'active';
    case 'ACKNOWLEDGED': return 'active';
    case 'FULFILLED': return 'success';
    case 'FAILED': return 'danger';
    case 'DECLINED': return 'neutral';
    case 'DELEGATED': return 'transfer';
    case 'EXPIRED': return 'warning';
  }
}
```

- [ ] **Step 6: Write types.test.ts**

```typescript
// chat-demo/src/main/webui/src/qhorus/types.test.ts
import { describe, it, expect } from 'vitest';
import {
  isTerminalMessageType, isObligationCreating,
  messageTypeCategory, commitmentStateCategory,
  MESSAGE_TYPES, ACTOR_TYPES, COMMITMENT_STATES,
} from './types.js';

describe('MessageType helpers', () => {
  it('identifies terminal types', () => {
    expect(isTerminalMessageType('DONE')).toBe(true);
    expect(isTerminalMessageType('FAILURE')).toBe(true);
    expect(isTerminalMessageType('DECLINE')).toBe(true);
    expect(isTerminalMessageType('HANDOFF')).toBe(true);
    expect(isTerminalMessageType('COMMAND')).toBe(false);
    expect(isTerminalMessageType('STATUS')).toBe(false);
  });

  it('identifies obligation-creating types', () => {
    expect(isObligationCreating('COMMAND')).toBe(true);
    expect(isObligationCreating('QUERY')).toBe(false);
    expect(isObligationCreating('EVENT')).toBe(false);
  });

  it('categorises every message type', () => {
    for (const type of MESSAGE_TYPES) {
      expect(messageTypeCategory(type)).toBeTruthy();
    }
    expect(messageTypeCategory('COMMAND')).toBe('obligation');
    expect(messageTypeCategory('DONE')).toBe('success');
    expect(messageTypeCategory('FAILURE')).toBe('danger');
    expect(messageTypeCategory('DECLINE')).toBe('warning');
    expect(messageTypeCategory('HANDOFF')).toBe('transfer');
    expect(messageTypeCategory('EVENT')).toBe('telemetry');
  });

  it('categorises every commitment state', () => {
    for (const state of COMMITMENT_STATES) {
      expect(commitmentStateCategory(state)).toBeTruthy();
    }
    expect(commitmentStateCategory('FULFILLED')).toBe('success');
    expect(commitmentStateCategory('FAILED')).toBe('danger');
    expect(commitmentStateCategory('DELEGATED')).toBe('transfer');
  });

  it('all enum arrays are non-empty', () => {
    expect(MESSAGE_TYPES.length).toBe(9);
    expect(ACTOR_TYPES.length).toBe(3);
    expect(COMMITMENT_STATES.length).toBe(7);
  });
});
```

- [ ] **Step 7: Run types tests**

```bash
cd chat-demo/src/main/webui && npx vitest run src/qhorus/types.test.ts
```

Expected: all tests pass.

- [ ] **Step 8: Write events.ts**

```typescript
// chat-demo/src/main/webui/src/qhorus/events.ts
import type { QhorusMessage, MessageType, ArtefactRef } from './types.js';

export const ChatEventTopics = {
  SEND_MESSAGE: 'chat:send-message',
  REACT: 'chat:react',
  UNREACT: 'chat:unreact',
  CREATE_CHANNEL: 'chat:create-channel',
  DELETE_CHANNEL: 'chat:delete-channel',
  SELECT_CHANNEL: 'chat:select-channel',
  SELECT_TOPIC: 'chat:select-topic',
  RESOLVE_TOPIC: 'chat:resolve-topic',
  MESSAGE_SELECTED: 'chat:message-selected',
} as const;

export interface SendMessagePayload {
  readonly channelId: string;
  readonly content: string;
  readonly topic?: string;
  readonly inReplyTo?: string;
  readonly speechAct?: MessageType;
  readonly artefactRefs?: readonly ArtefactRef[];
}

export interface ReactPayload {
  readonly messageId: string;
  readonly emoji: string;
}

export interface CreateChannelPayload {
  readonly name: string;
  readonly description?: string;
  readonly spaceId?: string;
  readonly semantic?: string;
}

export interface SelectChannelPayload {
  readonly channelId: string;
}

export interface MessageSelectedPayload {
  readonly message: QhorusMessage;
}

export function emitChatEvent<T>(
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

- [ ] **Step 9: Write events.test.ts**

```typescript
// chat-demo/src/main/webui/src/qhorus/events.test.ts
import { describe, it, expect, vi } from 'vitest';
import { emitChatEvent, ChatEventTopics } from './events.js';

describe('emitChatEvent', () => {
  it('dispatches pages-event with topic and payload', () => {
    const target = document.createElement('div');
    const handler = vi.fn();
    target.addEventListener('pages-event', handler);

    emitChatEvent(target, ChatEventTopics.SELECT_CHANNEL, { channelId: 'ch-1' });

    expect(handler).toHaveBeenCalledOnce();
    const event = handler.mock.calls[0][0] as CustomEvent;
    expect(event.detail.topic).toBe('chat:select-channel');
    expect(event.detail.payload).toEqual({ channelId: 'ch-1' });
  });

  it('bubbles and composes', () => {
    const target = document.createElement('div');
    const handler = vi.fn();
    document.body.appendChild(target);
    document.body.addEventListener('pages-event', handler);

    emitChatEvent(target, ChatEventTopics.SEND_MESSAGE, { channelId: 'ch-1', content: 'hello' });

    expect(handler).toHaveBeenCalledOnce();
    document.body.removeChild(target);
    document.body.removeEventListener('pages-event', handler);
  });

  it('all topic constants have chat: prefix', () => {
    for (const topic of Object.values(ChatEventTopics)) {
      expect(topic).toMatch(/^chat:/);
    }
  });
});
```

- [ ] **Step 10: Write markdown.ts**

```typescript
// chat-demo/src/main/webui/src/qhorus/markdown.ts
import { marked } from 'marked';
import DOMPurify from 'dompurify';

marked.setOptions({
  breaks: true,
  gfm: true,
});

export function renderMarkdown(content: string): string {
  const raw = marked.parse(content, { async: false }) as string;
  return DOMPurify.sanitize(raw, {
    ALLOWED_TAGS: [
      'p', 'br', 'strong', 'em', 'code', 'pre', 'a', 'ul', 'ol', 'li',
      'blockquote', 'h1', 'h2', 'h3', 'h4', 'h5', 'h6', 'hr', 'del',
      'table', 'thead', 'tbody', 'tr', 'th', 'td', 'img', 'span',
    ],
    ALLOWED_ATTR: ['href', 'title', 'alt', 'src', 'class', 'target'],
  });
}
```

- [ ] **Step 11: Write markdown.test.ts**

```typescript
// chat-demo/src/main/webui/src/qhorus/markdown.test.ts
import { describe, it, expect } from 'vitest';
import { renderMarkdown } from './markdown.js';

describe('renderMarkdown', () => {
  it('renders plain text as paragraph', () => {
    expect(renderMarkdown('hello world')).toContain('hello world');
  });

  it('renders bold and italic', () => {
    const html = renderMarkdown('**bold** and *italic*');
    expect(html).toContain('<strong>bold</strong>');
    expect(html).toContain('<em>italic</em>');
  });

  it('renders code blocks', () => {
    const html = renderMarkdown('```\nconst x = 1;\n```');
    expect(html).toContain('<code>');
  });

  it('renders inline code', () => {
    expect(renderMarkdown('use `foo()`')).toContain('<code>foo()</code>');
  });

  it('sanitises script tags', () => {
    const html = renderMarkdown('<script>alert("xss")</script>');
    expect(html).not.toContain('<script>');
  });

  it('sanitises event handlers', () => {
    const html = renderMarkdown('<img onerror="alert(1)" src="x">');
    expect(html).not.toContain('onerror');
  });

  it('allows links with href', () => {
    const html = renderMarkdown('[click](https://example.com)');
    expect(html).toContain('href="https://example.com"');
  });

  it('strips javascript: URIs', () => {
    const html = renderMarkdown('[xss](javascript:alert(1))');
    expect(html).not.toContain('javascript:');
  });
});
```

- [ ] **Step 12: Run all Task 1 tests**

```bash
cd chat-demo/src/main/webui && npx vitest run src/qhorus/
```

Expected: all tests pass (types, events, markdown).

- [ ] **Step 13: Commit**

```bash
git add -A chat-demo/src/main/webui/src/qhorus/ chat-demo/src/main/webui/package.json chat-demo/src/main/webui/tsconfig.json chat-demo/src/main/webui/vite.config.ts chat-demo/src/main/webui/vitest.config.ts
git commit -m "feat(chat-demo): scaffold qhorus UI — types, events, markdown renderer"
```

---

### Task 2: `<qhorus-message>` Primitive

**Files:**
- Create: `chat-demo/src/main/webui/src/qhorus/primitives/qhorus-message.ts`
- Create: `chat-demo/src/main/webui/src/qhorus/primitives/qhorus-message.test.ts`

**Interfaces:**
- Consumes: `QhorusMessage`, `Reaction`, `MessageType`, `ActorType`, `CommitmentState`, `messageTypeCategory`, `commitmentStateCategory`, `renderMarkdown`, `emitChatEvent`, `ChatEventTopics` from Task 1
- Produces: `<qhorus-message>` custom element. Properties: `message: QhorusMessage`, `reactions: Reaction[]`, `showSpeechAct: boolean`, `showActorBadge: boolean`, `expanded: boolean`. Emits: `chat:message-selected`

- [ ] **Step 1: Write failing test — renders sender and content**

```typescript
// chat-demo/src/main/webui/src/qhorus/primitives/qhorus-message.test.ts
import { describe, it, expect, beforeEach, afterEach } from 'vitest';
import './qhorus-message.js';
import type { QhorusMessage } from '../types.js';

function makeMessage(overrides: Partial<QhorusMessage> = {}): QhorusMessage {
  return {
    id: 'msg-1',
    channelId: 'ch-1',
    sender: 'agent-alpha',
    messageType: 'EVENT',
    actorType: 'AGENT',
    content: 'Hello world',
    replyCount: 0,
    artefactRefs: [],
    createdAt: '2026-07-07T12:00:00Z',
    ...overrides,
  };
}

async function renderMessage(props: Record<string, unknown> = {}): Promise<HTMLElement> {
  const el = document.createElement('qhorus-message') as any;
  el.message = makeMessage(props.message as any);
  if (props.reactions) el.reactions = props.reactions;
  if (props.showSpeechAct !== undefined) el.showSpeechAct = props.showSpeechAct;
  if (props.showActorBadge !== undefined) el.showActorBadge = props.showActorBadge;
  if (props.expanded !== undefined) el.expanded = props.expanded;
  document.body.appendChild(el);
  await el.updateComplete;
  return el;
}

afterEach(() => {
  document.body.innerHTML = '';
});

describe('qhorus-message', () => {
  it('renders sender name', async () => {
    const el = await renderMessage();
    const shadow = el.shadowRoot!;
    expect(shadow.textContent).toContain('agent-alpha');
  });

  it('renders message content as markdown', async () => {
    const el = await renderMessage({ message: { content: '**bold** text' } });
    const shadow = el.shadowRoot!;
    expect(shadow.innerHTML).toContain('<strong>bold</strong>');
  });

  it('renders speech act badge by default', async () => {
    const el = await renderMessage({ message: { messageType: 'COMMAND' } });
    const badge = el.shadowRoot!.querySelector('.speech-act-badge');
    expect(badge).toBeTruthy();
    expect(badge!.textContent!.trim()).toBe('COMMAND');
  });

  it('hides speech act badge when showSpeechAct=false', async () => {
    const el = await renderMessage({ showSpeechAct: false });
    const badge = el.shadowRoot!.querySelector('.speech-act-badge');
    expect(badge).toBeNull();
  });

  it('renders actor icon for AGENT type', async () => {
    const el = await renderMessage({ message: { actorType: 'AGENT' } });
    const icon = el.shadowRoot!.querySelector('.actor-icon');
    expect(icon).toBeTruthy();
    expect(icon!.getAttribute('data-actor')).toBe('AGENT');
  });

  it('renders actor icon for HUMAN type', async () => {
    const el = await renderMessage({ message: { actorType: 'HUMAN' } });
    const icon = el.shadowRoot!.querySelector('.actor-icon');
    expect(icon!.getAttribute('data-actor')).toBe('HUMAN');
  });

  it('renders commitment state badge for COMMAND messages', async () => {
    const el = await renderMessage({
      message: { messageType: 'COMMAND', commitmentId: 'c-1' },
    });
    el.commitmentState = 'OPEN';
    await (el as any).updateComplete;
    const badge = el.shadowRoot!.querySelector('.commitment-badge');
    expect(badge).toBeTruthy();
    expect(badge!.textContent!.trim()).toBe('OPEN');
  });

  it('renders relative timestamp', async () => {
    const el = await renderMessage();
    const time = el.shadowRoot!.querySelector('time');
    expect(time).toBeTruthy();
    expect(time!.getAttribute('datetime')).toBe('2026-07-07T12:00:00Z');
  });

  it('applies correct badge color class for each message type', async () => {
    for (const [type, expected] of [
      ['COMMAND', 'obligation'],
      ['DONE', 'success'],
      ['FAILURE', 'danger'],
      ['DECLINE', 'warning'],
      ['HANDOFF', 'transfer'],
      ['EVENT', 'telemetry'],
      ['QUERY', 'info'],
    ] as const) {
      const el = await renderMessage({ message: { messageType: type } });
      const badge = el.shadowRoot!.querySelector('.speech-act-badge');
      expect(badge!.classList.contains(`badge-${expected}`),
        `${type} should have badge-${expected}`).toBe(true);
      document.body.innerHTML = '';
    }
  });

  it('renders delegation indicator for HANDOFF messages', async () => {
    const el = await renderMessage({
      message: { messageType: 'HANDOFF', target: 'agent-beta', sender: 'agent-alpha' },
    });
    const delegation = el.shadowRoot!.querySelector('.delegation-indicator');
    expect(delegation).toBeTruthy();
    expect(delegation!.textContent).toContain('agent-beta');
  });

  it('renders artefact chips when artefactRefs present', async () => {
    const el = await renderMessage({
      message: {
        artefactRefs: [
          { uri: 'doc:spec.md', type: 'DOCUMENT', label: 'Design Spec' },
        ],
      },
    });
    const chip = el.shadowRoot!.querySelector('.artefact-chip');
    expect(chip).toBeTruthy();
    expect(chip!.textContent).toContain('Design Spec');
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

```bash
cd chat-demo/src/main/webui && npx vitest run src/qhorus/primitives/qhorus-message.test.ts
```

Expected: FAIL — module not found.

- [ ] **Step 3: Implement qhorus-message.ts**

```typescript
// chat-demo/src/main/webui/src/qhorus/primitives/qhorus-message.ts
import { LitElement, html, css, nothing } from 'lit';
import { customElement, property, state } from 'lit/decorators.js';
import { unsafeHTML } from 'lit/directives/unsafe-html.js';
import type { QhorusMessage, Reaction, CommitmentState } from '../types.js';
import { messageTypeCategory, commitmentStateCategory, isObligationCreating } from '../types.js';
import { renderMarkdown } from '../markdown.js';
import { emitChatEvent, ChatEventTopics } from '../events.js';

@customElement('qhorus-message')
export class QhorusMessageElement extends LitElement {
  @property({ type: Object }) message!: QhorusMessage;
  @property({ type: Array }) reactions: Reaction[] = [];
  @property({ type: Boolean }) showSpeechAct = true;
  @property({ type: Boolean }) showActorBadge = true;
  @property({ type: Boolean }) expanded = false;
  @property({ type: String }) commitmentState?: CommitmentState;

  static override readonly styles = css`
    :host {
      display: block;
      padding: var(--pages-space-2, 8px) var(--pages-space-4, 16px);
    }
    :host(:hover) {
      background: var(--pages-neutral-2, #f5f5f5);
    }
    .message-header {
      display: flex;
      align-items: center;
      gap: var(--pages-space-2, 8px);
      margin-bottom: var(--pages-space-1, 4px);
    }
    .actor-icon {
      width: 20px;
      height: 20px;
      border-radius: 50%;
      display: flex;
      align-items: center;
      justify-content: center;
      font-size: var(--pages-font-size-xs, 11px);
      background: var(--pages-neutral-4, #ddd);
      flex-shrink: 0;
    }
    .actor-icon[data-actor="AGENT"] { background: var(--pages-accent-3, #cce5ff); }
    .actor-icon[data-actor="SYSTEM"] { background: var(--pages-neutral-5, #ccc); }
    .sender {
      font-weight: var(--pages-font-weight-semibold, 600);
      font-size: var(--pages-font-size-sm, 13px);
      color: var(--pages-neutral-12, #111);
    }
    time {
      font-size: var(--pages-font-size-xs, 11px);
      color: var(--pages-neutral-8, #888);
    }
    .speech-act-badge {
      font-size: 10px;
      font-weight: var(--pages-font-weight-medium, 500);
      padding: 1px 6px;
      border-radius: 9999px;
      text-transform: uppercase;
      letter-spacing: 0.5px;
    }
    .badge-info { background: var(--pages-info-3, #dbeafe); color: var(--pages-info-11, #1e40af); }
    .badge-obligation { background: var(--pages-accent-3, #e0e7ff); color: var(--pages-accent-11, #3730a3); }
    .badge-success { background: var(--pages-success-3, #d1fae5); color: var(--pages-success-11, #065f46); }
    .badge-danger { background: var(--pages-danger-3, #fee2e2); color: var(--pages-danger-11, #991b1b); }
    .badge-warning { background: var(--pages-warning-3, #fef3c7); color: var(--pages-warning-11, #92400e); }
    .badge-transfer { background: var(--pages-info-3, #dbeafe); color: var(--pages-info-11, #1e40af); }
    .badge-telemetry { background: var(--pages-neutral-3, #e5e5e5); color: var(--pages-neutral-9, #737373); }
    .commitment-badge {
      font-size: 10px;
      padding: 1px 6px;
      border-radius: var(--pages-radius-sm, 4px);
    }
    .commitment-active { background: var(--pages-accent-3); color: var(--pages-accent-11); }
    .commitment-success { background: var(--pages-success-3); color: var(--pages-success-11); }
    .commitment-danger { background: var(--pages-danger-3); color: var(--pages-danger-11); }
    .commitment-neutral { background: var(--pages-neutral-3); color: var(--pages-neutral-9); }
    .commitment-transfer { background: var(--pages-info-3); color: var(--pages-info-11); }
    .commitment-warning { background: var(--pages-warning-3); color: var(--pages-warning-11); }
    .content {
      font-size: var(--pages-font-size-base, 14px);
      line-height: var(--pages-line-height-base, 20px);
      color: var(--pages-neutral-11, #333);
    }
    .content :first-child { margin-top: 0; }
    .content :last-child { margin-bottom: 0; }
    .delegation-indicator {
      display: flex;
      align-items: center;
      gap: var(--pages-space-1, 4px);
      font-size: var(--pages-font-size-xs, 11px);
      color: var(--pages-info-9, #2563eb);
      margin-top: var(--pages-space-1, 4px);
    }
    .artefact-chip {
      display: inline-flex;
      align-items: center;
      gap: var(--pages-space-1, 4px);
      font-size: var(--pages-font-size-xs, 11px);
      padding: 2px 8px;
      border-radius: var(--pages-radius-sm, 4px);
      background: var(--pages-neutral-2, #f5f5f5);
      border: 1px solid var(--pages-neutral-5, #d4d4d4);
      cursor: pointer;
      margin-top: var(--pages-space-1, 4px);
      margin-right: var(--pages-space-1, 4px);
    }
    .artefact-chip:hover { background: var(--pages-neutral-3, #e5e5e5); }
  `;

  private _formatTime(iso: string): string {
    const date = new Date(iso);
    const now = new Date();
    const diffMs = now.getTime() - date.getTime();
    const diffMin = Math.floor(diffMs / 60000);
    if (diffMin < 1) return 'now';
    if (diffMin < 60) return `${diffMin}m`;
    const diffHr = Math.floor(diffMin / 60);
    if (diffHr < 24) return `${diffHr}h`;
    return `${Math.floor(diffHr / 24)}d`;
  }

  private _actorIcon(type: string): string {
    switch (type) {
      case 'HUMAN': return '\u{1F464}';
      case 'AGENT': return '\u{1F916}';
      case 'SYSTEM': return '⚙';
      default: return '?';
    }
  }

  override render() {
    if (!this.message) return nothing;
    const m = this.message;
    const category = messageTypeCategory(m.messageType);

    return html`
      <div class="message-header">
        ${this.showActorBadge ? html`
          <span class="actor-icon" data-actor=${m.actorType}>${this._actorIcon(m.actorType)}</span>
        ` : nothing}
        <span class="sender">${m.sender}</span>
        ${this.showSpeechAct ? html`
          <span class="speech-act-badge badge-${category}">${m.messageType}</span>
        ` : nothing}
        ${this.commitmentState && isObligationCreating(m.messageType) ? html`
          <span class="commitment-badge commitment-${commitmentStateCategory(this.commitmentState)}">${this.commitmentState}</span>
        ` : nothing}
        <time datetime=${m.createdAt}>${this._formatTime(m.createdAt)}</time>
      </div>
      <div class="content">${unsafeHTML(renderMarkdown(m.content))}</div>
      ${m.messageType === 'HANDOFF' && m.target ? html`
        <div class="delegation-indicator">
          ↳ Delegated to <strong>${m.target}</strong>
        </div>
      ` : nothing}
      ${m.artefactRefs.length > 0 ? html`
        <div class="artefact-chips">
          ${m.artefactRefs.map(ref => html`
            <span class="artefact-chip" data-type=${ref.type}>${ref.label}</span>
          `)}
        </div>
      ` : nothing}
      ${this.reactions.length > 0 ? html`
        <qhorus-reaction-bar .reactions=${this.reactions}></qhorus-reaction-bar>
      ` : nothing}
    `;
  }
}

declare global {
  interface HTMLElementTagNameMap {
    'qhorus-message': QhorusMessageElement;
  }
}
```

- [ ] **Step 4: Run tests**

```bash
cd chat-demo/src/main/webui && npx vitest run src/qhorus/primitives/qhorus-message.test.ts
```

Expected: all tests pass.

- [ ] **Step 5: Commit**

```bash
git add chat-demo/src/main/webui/src/qhorus/primitives/qhorus-message.ts chat-demo/src/main/webui/src/qhorus/primitives/qhorus-message.test.ts
git commit -m "feat(chat-demo): qhorus-message primitive — speech acts, actor icons, commitment badges"
```

---

### Task 3: `<qhorus-reaction-bar>` and `<qhorus-thread>` Primitives

**Files:**
- Create: `chat-demo/src/main/webui/src/qhorus/primitives/qhorus-reaction-bar.ts`
- Create: `chat-demo/src/main/webui/src/qhorus/primitives/qhorus-reaction-bar.test.ts`
- Create: `chat-demo/src/main/webui/src/qhorus/primitives/qhorus-thread.ts`
- Create: `chat-demo/src/main/webui/src/qhorus/primitives/qhorus-thread.test.ts`

**Interfaces:**
- Consumes: `Reaction`, `QhorusMessage`, `CommitmentState` from Task 1, `<qhorus-message>` from Task 2
- Produces:
  - `<qhorus-reaction-bar>` — Properties: `reactions: Reaction[]`, `currentActorId?: string`. Emits: `chat:react`, `chat:unreact`
  - `<qhorus-thread>` — Properties: `rootMessage: QhorusMessage`, `replies: QhorusMessage[]`, `collapsed: boolean`, `commitmentState?: CommitmentState`. No events (delegates to child `<qhorus-message>`)

- [ ] **Step 1: Write failing reaction-bar tests**

```typescript
// chat-demo/src/main/webui/src/qhorus/primitives/qhorus-reaction-bar.test.ts
import { describe, it, expect, vi, afterEach } from 'vitest';
import './qhorus-reaction-bar.js';
import type { Reaction } from '../types.js';

function makeReactions(specs: Array<[string, string[]]>): Reaction[] {
  return specs.flatMap(([emoji, actors]) =>
    actors.map(actorId => ({ messageId: 'msg-1', emoji, actorId, createdAt: '2026-07-07T12:00:00Z' }))
  );
}

afterEach(() => { document.body.innerHTML = ''; });

describe('qhorus-reaction-bar', () => {
  it('renders grouped reaction pills', async () => {
    const el = document.createElement('qhorus-reaction-bar') as any;
    el.reactions = makeReactions([['👍', ['a', 'b']], ['❤️', ['a']]]);
    el.messageId = 'msg-1';
    document.body.appendChild(el);
    await el.updateComplete;

    const pills = el.shadowRoot!.querySelectorAll('.reaction-pill');
    expect(pills.length).toBe(2);
    expect(pills[0].textContent).toContain('👍');
    expect(pills[0].textContent).toContain('2');
    expect(pills[1].textContent).toContain('❤️');
    expect(pills[1].textContent).toContain('1');
  });

  it('highlights pills where current user reacted', async () => {
    const el = document.createElement('qhorus-reaction-bar') as any;
    el.reactions = makeReactions([['👍', ['me', 'other']]]);
    el.messageId = 'msg-1';
    el.currentActorId = 'me';
    document.body.appendChild(el);
    await el.updateComplete;

    const pill = el.shadowRoot!.querySelector('.reaction-pill');
    expect(pill!.classList.contains('reacted')).toBe(true);
  });

  it('emits chat:react on click when not reacted', async () => {
    const el = document.createElement('qhorus-reaction-bar') as any;
    el.reactions = makeReactions([['👍', ['other']]]);
    el.messageId = 'msg-1';
    el.currentActorId = 'me';
    document.body.appendChild(el);
    await el.updateComplete;

    const handler = vi.fn();
    el.addEventListener('pages-event', handler);
    el.shadowRoot!.querySelector('.reaction-pill')!.click();

    expect(handler).toHaveBeenCalledOnce();
    expect(handler.mock.calls[0][0].detail.topic).toBe('chat:react');
    expect(handler.mock.calls[0][0].detail.payload).toEqual({ messageId: 'msg-1', emoji: '👍' });
  });

  it('emits chat:unreact on click when already reacted', async () => {
    const el = document.createElement('qhorus-reaction-bar') as any;
    el.reactions = makeReactions([['👍', ['me']]]);
    el.messageId = 'msg-1';
    el.currentActorId = 'me';
    document.body.appendChild(el);
    await el.updateComplete;

    const handler = vi.fn();
    el.addEventListener('pages-event', handler);
    el.shadowRoot!.querySelector('.reaction-pill')!.click();

    expect(handler.mock.calls[0][0].detail.topic).toBe('chat:unreact');
  });
});
```

- [ ] **Step 2: Implement qhorus-reaction-bar.ts**

```typescript
// chat-demo/src/main/webui/src/qhorus/primitives/qhorus-reaction-bar.ts
import { LitElement, html, css, nothing } from 'lit';
import { customElement, property } from 'lit/decorators.js';
import type { Reaction } from '../types.js';
import { emitChatEvent, ChatEventTopics } from '../events.js';

interface GroupedReaction {
  readonly emoji: string;
  readonly count: number;
  readonly actors: readonly string[];
  readonly userReacted: boolean;
}

@customElement('qhorus-reaction-bar')
export class QhorusReactionBarElement extends LitElement {
  @property({ type: Array }) reactions: Reaction[] = [];
  @property({ type: String }) messageId = '';
  @property({ type: String }) currentActorId?: string;

  static override readonly styles = css`
    :host { display: flex; gap: var(--pages-space-1, 4px); flex-wrap: wrap; margin-top: var(--pages-space-1, 4px); }
    .reaction-pill {
      display: inline-flex;
      align-items: center;
      gap: 3px;
      padding: 2px 8px;
      border-radius: 9999px;
      border: 1px solid var(--pages-neutral-5, #d4d4d4);
      background: var(--pages-neutral-1, #fafafa);
      font-size: var(--pages-font-size-xs, 11px);
      cursor: pointer;
      user-select: none;
    }
    .reaction-pill:hover { background: var(--pages-neutral-3, #e5e5e5); }
    .reaction-pill.reacted {
      border-color: var(--pages-accent-7, #818cf8);
      background: var(--pages-accent-2, #eef2ff);
    }
    .count { color: var(--pages-neutral-9, #737373); }
  `;

  private _grouped(): GroupedReaction[] {
    const map = new Map<string, { actors: string[] }>();
    for (const r of this.reactions) {
      const entry = map.get(r.emoji) ?? { actors: [] };
      entry.actors.push(r.actorId);
      map.set(r.emoji, entry);
    }
    return [...map.entries()].map(([emoji, { actors }]) => ({
      emoji,
      count: actors.length,
      actors,
      userReacted: this.currentActorId != null && actors.includes(this.currentActorId),
    }));
  }

  private _toggleReaction(emoji: string, userReacted: boolean) {
    const topic = userReacted ? ChatEventTopics.UNREACT : ChatEventTopics.REACT;
    emitChatEvent(this, topic, { messageId: this.messageId, emoji });
  }

  override render() {
    const groups = this._grouped();
    if (groups.length === 0) return nothing;
    return html`${groups.map(g => html`
      <button class="reaction-pill ${g.userReacted ? 'reacted' : ''}"
              @click=${() => this._toggleReaction(g.emoji, g.userReacted)}>
        <span class="emoji">${g.emoji}</span>
        <span class="count">${g.count}</span>
      </button>
    `)}`;
  }
}

declare global {
  interface HTMLElementTagNameMap {
    'qhorus-reaction-bar': QhorusReactionBarElement;
  }
}
```

- [ ] **Step 3: Run reaction-bar tests**

```bash
cd chat-demo/src/main/webui && npx vitest run src/qhorus/primitives/qhorus-reaction-bar.test.ts
```

Expected: all pass.

- [ ] **Step 4: Write failing thread tests**

```typescript
// chat-demo/src/main/webui/src/qhorus/primitives/qhorus-thread.test.ts
import { describe, it, expect, afterEach } from 'vitest';
import './qhorus-thread.js';
import '../primitives/qhorus-message.js';
import type { QhorusMessage } from '../types.js';

function msg(id: string, type: string, content: string): QhorusMessage {
  return {
    id, channelId: 'ch-1', sender: 'agent-a', messageType: type as any,
    actorType: 'AGENT', content, replyCount: 0, artefactRefs: [],
    createdAt: '2026-07-07T12:00:00Z',
  };
}

afterEach(() => { document.body.innerHTML = ''; });

describe('qhorus-thread', () => {
  it('renders root message and reply count when collapsed', async () => {
    const el = document.createElement('qhorus-thread') as any;
    el.rootMessage = msg('1', 'COMMAND', 'Analyze auth');
    el.replies = [msg('2', 'STATUS', 'Reading files'), msg('3', 'DONE', 'Complete')];
    el.collapsed = true;
    document.body.appendChild(el);
    await el.updateComplete;

    const shadow = el.shadowRoot!;
    expect(shadow.querySelector('qhorus-message')).toBeTruthy();
    expect(shadow.textContent).toContain('2 replies');
    expect(shadow.querySelectorAll('.reply qhorus-message').length).toBe(0);
  });

  it('renders all replies when expanded', async () => {
    const el = document.createElement('qhorus-thread') as any;
    el.rootMessage = msg('1', 'COMMAND', 'Analyze auth');
    el.replies = [msg('2', 'STATUS', 'Reading'), msg('3', 'DONE', 'Done')];
    el.collapsed = false;
    document.body.appendChild(el);
    await el.updateComplete;

    const messages = el.shadowRoot!.querySelectorAll('qhorus-message');
    expect(messages.length).toBe(3);
  });

  it('toggles collapse on header click', async () => {
    const el = document.createElement('qhorus-thread') as any;
    el.rootMessage = msg('1', 'COMMAND', 'Task');
    el.replies = [msg('2', 'DONE', 'Done')];
    el.collapsed = true;
    document.body.appendChild(el);
    await el.updateComplete;

    el.shadowRoot!.querySelector('.thread-toggle')!.click();
    await el.updateComplete;

    expect(el.collapsed).toBe(false);
    expect(el.shadowRoot!.querySelectorAll('qhorus-message').length).toBe(2);
  });

  it('shows commitment state on header', async () => {
    const el = document.createElement('qhorus-thread') as any;
    el.rootMessage = msg('1', 'COMMAND', 'Task');
    el.replies = [];
    el.commitmentState = 'FULFILLED';
    document.body.appendChild(el);
    await el.updateComplete;

    const badge = el.shadowRoot!.querySelector('.thread-commitment');
    expect(badge).toBeTruthy();
    expect(badge!.textContent).toContain('FULFILLED');
  });
});
```

- [ ] **Step 5: Implement qhorus-thread.ts**

```typescript
// chat-demo/src/main/webui/src/qhorus/primitives/qhorus-thread.ts
import { LitElement, html, css, nothing } from 'lit';
import { customElement, property } from 'lit/decorators.js';
import type { QhorusMessage, CommitmentState } from '../types.js';
import { commitmentStateCategory } from '../types.js';
import './qhorus-message.js';

@customElement('qhorus-thread')
export class QhorusThreadElement extends LitElement {
  @property({ type: Object }) rootMessage!: QhorusMessage;
  @property({ type: Array }) replies: QhorusMessage[] = [];
  @property({ type: Boolean }) collapsed = true;
  @property({ type: String }) commitmentState?: CommitmentState;

  static override readonly styles = css`
    :host {
      display: block;
      border-left: 2px solid var(--pages-neutral-5, #d4d4d4);
      margin: var(--pages-space-2, 8px) 0;
      border-radius: var(--pages-radius-sm, 4px);
    }
    .thread-header {
      display: flex;
      align-items: center;
      gap: var(--pages-space-2, 8px);
      padding: var(--pages-space-1, 4px) var(--pages-space-2, 8px);
    }
    .thread-toggle {
      font-size: var(--pages-font-size-xs, 11px);
      color: var(--pages-accent-9, #6366f1);
      cursor: pointer;
      background: none;
      border: none;
      padding: 2px 6px;
      border-radius: var(--pages-radius-sm, 4px);
    }
    .thread-toggle:hover { background: var(--pages-neutral-3, #e5e5e5); }
    .thread-commitment {
      font-size: 10px;
      padding: 1px 6px;
      border-radius: var(--pages-radius-sm, 4px);
    }
    .reply { padding-left: var(--pages-space-4, 16px); }
  `;

  private _toggle() {
    this.collapsed = !this.collapsed;
  }

  private _summary(): string {
    const count = this.replies.length;
    if (count === 0) return 'no replies';
    return `${count} ${count === 1 ? 'reply' : 'replies'}`;
  }

  override render() {
    if (!this.rootMessage) return nothing;

    return html`
      <qhorus-message .message=${this.rootMessage}
                      .commitmentState=${this.commitmentState}></qhorus-message>
      ${this.replies.length > 0 ? html`
        <div class="thread-header">
          <button class="thread-toggle"
                  @click=${this._toggle}
                  aria-expanded=${!this.collapsed}>
            ${this.collapsed ? '▶' : '▼'} ${this._summary()}
          </button>
          ${this.commitmentState ? html`
            <span class="thread-commitment commitment-${commitmentStateCategory(this.commitmentState)}">
              ${this.commitmentState}
            </span>
          ` : nothing}
        </div>
        ${!this.collapsed ? html`
          ${this.replies.map(r => html`
            <div class="reply">
              <qhorus-message .message=${r}></qhorus-message>
            </div>
          `)}
        ` : nothing}
      ` : nothing}
    `;
  }
}

declare global {
  interface HTMLElementTagNameMap {
    'qhorus-thread': QhorusThreadElement;
  }
}
```

- [ ] **Step 6: Run all Task 3 tests**

```bash
cd chat-demo/src/main/webui && npx vitest run src/qhorus/primitives/qhorus-reaction-bar.test.ts src/qhorus/primitives/qhorus-thread.test.ts
```

Expected: all pass.

- [ ] **Step 7: Commit**

```bash
git add chat-demo/src/main/webui/src/qhorus/primitives/qhorus-reaction-bar.ts chat-demo/src/main/webui/src/qhorus/primitives/qhorus-reaction-bar.test.ts chat-demo/src/main/webui/src/qhorus/primitives/qhorus-thread.ts chat-demo/src/main/webui/src/qhorus/primitives/qhorus-thread.test.ts
git commit -m "feat(chat-demo): reaction-bar and thread primitives"
```

---

### Task 4: `<qhorus-message-input>` Primitive

**Files:**
- Create: `chat-demo/src/main/webui/src/qhorus/primitives/qhorus-message-input.ts`
- Create: `chat-demo/src/main/webui/src/qhorus/primitives/qhorus-message-input.test.ts`

**Interfaces:**
- Consumes: `emitChatEvent`, `ChatEventTopics`, `SendMessagePayload` from Task 1
- Produces: `<qhorus-message-input>` — Properties: `channelId: string`, `replyTo?: {messageId: string, senderName: string}`. Emits: `chat:send-message`

- [ ] **Step 1: Write failing tests**

```typescript
// chat-demo/src/main/webui/src/qhorus/primitives/qhorus-message-input.test.ts
import { describe, it, expect, vi, afterEach } from 'vitest';
import './qhorus-message-input.js';

afterEach(() => { document.body.innerHTML = ''; });

describe('qhorus-message-input', () => {
  it('renders a textarea', async () => {
    const el = document.createElement('qhorus-message-input') as any;
    el.channelId = 'ch-1';
    document.body.appendChild(el);
    await el.updateComplete;
    expect(el.shadowRoot!.querySelector('textarea')).toBeTruthy();
  });

  it('emits chat:send-message on Enter', async () => {
    const el = document.createElement('qhorus-message-input') as any;
    el.channelId = 'ch-1';
    document.body.appendChild(el);
    await el.updateComplete;

    const handler = vi.fn();
    el.addEventListener('pages-event', handler);
    const textarea = el.shadowRoot!.querySelector('textarea')!;
    textarea.value = 'hello world';
    textarea.dispatchEvent(new Event('input'));
    textarea.dispatchEvent(new KeyboardEvent('keydown', { key: 'Enter' }));

    expect(handler).toHaveBeenCalledOnce();
    const detail = handler.mock.calls[0][0].detail;
    expect(detail.topic).toBe('chat:send-message');
    expect(detail.payload.content).toBe('hello world');
    expect(detail.payload.channelId).toBe('ch-1');
  });

  it('does not emit on Shift+Enter (newline)', async () => {
    const el = document.createElement('qhorus-message-input') as any;
    el.channelId = 'ch-1';
    document.body.appendChild(el);
    await el.updateComplete;

    const handler = vi.fn();
    el.addEventListener('pages-event', handler);
    const textarea = el.shadowRoot!.querySelector('textarea')!;
    textarea.value = 'line one';
    textarea.dispatchEvent(new KeyboardEvent('keydown', { key: 'Enter', shiftKey: true }));

    expect(handler).not.toHaveBeenCalled();
  });

  it('does not emit empty messages', async () => {
    const el = document.createElement('qhorus-message-input') as any;
    el.channelId = 'ch-1';
    document.body.appendChild(el);
    await el.updateComplete;

    const handler = vi.fn();
    el.addEventListener('pages-event', handler);
    const textarea = el.shadowRoot!.querySelector('textarea')!;
    textarea.value = '   ';
    textarea.dispatchEvent(new Event('input'));
    textarea.dispatchEvent(new KeyboardEvent('keydown', { key: 'Enter' }));

    expect(handler).not.toHaveBeenCalled();
  });

  it('clears textarea after sending', async () => {
    const el = document.createElement('qhorus-message-input') as any;
    el.channelId = 'ch-1';
    document.body.appendChild(el);
    await el.updateComplete;

    const textarea = el.shadowRoot!.querySelector('textarea')!;
    textarea.value = 'hello';
    textarea.dispatchEvent(new Event('input'));
    textarea.dispatchEvent(new KeyboardEvent('keydown', { key: 'Enter' }));
    await el.updateComplete;

    expect(textarea.value).toBe('');
  });

  it('shows reply banner when replyTo is set', async () => {
    const el = document.createElement('qhorus-message-input') as any;
    el.channelId = 'ch-1';
    el.replyTo = { messageId: 'msg-1', senderName: 'agent-alpha' };
    document.body.appendChild(el);
    await el.updateComplete;

    const banner = el.shadowRoot!.querySelector('.reply-banner');
    expect(banner).toBeTruthy();
    expect(banner!.textContent).toContain('agent-alpha');
  });

  it('includes inReplyTo in sent message when replying', async () => {
    const el = document.createElement('qhorus-message-input') as any;
    el.channelId = 'ch-1';
    el.replyTo = { messageId: 'msg-1', senderName: 'alpha' };
    document.body.appendChild(el);
    await el.updateComplete;

    const handler = vi.fn();
    el.addEventListener('pages-event', handler);
    const textarea = el.shadowRoot!.querySelector('textarea')!;
    textarea.value = 'reply text';
    textarea.dispatchEvent(new Event('input'));
    textarea.dispatchEvent(new KeyboardEvent('keydown', { key: 'Enter' }));

    expect(handler.mock.calls[0][0].detail.payload.inReplyTo).toBe('msg-1');
  });
});
```

- [ ] **Step 2: Implement qhorus-message-input.ts**

```typescript
// chat-demo/src/main/webui/src/qhorus/primitives/qhorus-message-input.ts
import { LitElement, html, css, nothing } from 'lit';
import { customElement, property, state, query } from 'lit/decorators.js';
import { emitChatEvent, ChatEventTopics } from '../events.js';

@customElement('qhorus-message-input')
export class QhorusMessageInputElement extends LitElement {
  @property({ type: String }) channelId = '';
  @property({ type: Object }) replyTo?: { messageId: string; senderName: string };

  @state() private _text = '';

  @query('textarea') private _textarea!: HTMLTextAreaElement;

  static override readonly styles = css`
    :host {
      display: block;
      padding: var(--pages-space-2, 8px) var(--pages-space-4, 16px);
      border-top: 1px solid var(--pages-neutral-4, #e5e5e5);
    }
    .reply-banner {
      display: flex;
      align-items: center;
      justify-content: space-between;
      padding: var(--pages-space-1, 4px) var(--pages-space-2, 8px);
      background: var(--pages-accent-2, #eef2ff);
      border-radius: var(--pages-radius-sm, 4px);
      margin-bottom: var(--pages-space-2, 8px);
      font-size: var(--pages-font-size-xs, 11px);
      color: var(--pages-accent-11, #3730a3);
    }
    .reply-cancel {
      cursor: pointer;
      background: none;
      border: none;
      color: var(--pages-neutral-8, #888);
      font-size: 14px;
    }
    textarea {
      width: 100%;
      resize: none;
      border: 1px solid var(--pages-neutral-5, #d4d4d4);
      border-radius: var(--pages-radius-md, 6px);
      padding: var(--pages-space-2, 8px) var(--pages-space-3, 12px);
      font-family: var(--pages-font-family, 'Inter', system-ui, sans-serif);
      font-size: var(--pages-font-size-base, 14px);
      line-height: var(--pages-line-height-base, 20px);
      color: var(--pages-neutral-12, #111);
      background: var(--pages-neutral-1, #fafafa);
      min-height: 40px;
      max-height: 200px;
      overflow-y: auto;
      box-sizing: border-box;
    }
    textarea:focus {
      outline: none;
      border-color: var(--pages-accent-7, #818cf8);
      box-shadow: 0 0 0 2px var(--pages-accent-3, #e0e7ff);
    }
  `;

  private _handleKeydown(e: KeyboardEvent) {
    if (e.key === 'Enter' && !e.shiftKey) {
      e.preventDefault();
      this._send();
    }
  }

  private _handleInput() {
    this._text = this._textarea.value;
    this._autoResize();
  }

  private _autoResize() {
    const ta = this._textarea;
    ta.style.height = 'auto';
    ta.style.height = `${Math.min(ta.scrollHeight, 200)}px`;
  }

  private _send() {
    const content = this._text.trim();
    if (!content || !this.channelId) return;

    emitChatEvent(this, ChatEventTopics.SEND_MESSAGE, {
      channelId: this.channelId,
      content,
      ...(this.replyTo ? { inReplyTo: this.replyTo.messageId } : {}),
    });

    this._text = '';
    this._textarea.value = '';
    this._textarea.style.height = 'auto';
    this.replyTo = undefined;
  }

  private _cancelReply() {
    this.replyTo = undefined;
  }

  override render() {
    return html`
      ${this.replyTo ? html`
        <div class="reply-banner">
          <span>Replying to <strong>${this.replyTo.senderName}</strong></span>
          <button class="reply-cancel" @click=${this._cancelReply}>✕</button>
        </div>
      ` : nothing}
      <textarea
        placeholder="Type a message..."
        @keydown=${this._handleKeydown}
        @input=${this._handleInput}
        rows="1"
      ></textarea>
    `;
  }
}

declare global {
  interface HTMLElementTagNameMap {
    'qhorus-message-input': QhorusMessageInputElement;
  }
}
```

- [ ] **Step 3: Run tests and commit**

```bash
cd chat-demo/src/main/webui && npx vitest run src/qhorus/primitives/qhorus-message-input.test.ts
```

```bash
git add chat-demo/src/main/webui/src/qhorus/primitives/qhorus-message-input.ts chat-demo/src/main/webui/src/qhorus/primitives/qhorus-message-input.test.ts
git commit -m "feat(chat-demo): message-input primitive — auto-resize, reply banner, Enter to send"
```

---

### Task 5: `<qhorus-channel-feed>` Composite

**Files:**
- Create: `chat-demo/src/main/webui/src/qhorus/composites/qhorus-channel-feed.ts`
- Create: `chat-demo/src/main/webui/src/qhorus/composites/qhorus-channel-feed.test.ts`

**Interfaces:**
- Consumes: `QhorusMessage`, `Reaction`, `CommitmentState` from Task 1; `<qhorus-message>` from Task 2; `<qhorus-thread>` from Task 3
- Produces: `<qhorus-channel-feed>` — Properties: `messages: QhorusMessage[]`, `reactions: Reaction[]`, `commitments: Map<string, CommitmentState>`, `mode: 'flat' | 'threaded'`. Emits: `chat:message-selected`

This is the core rendering engine. Flat mode renders messages chronologically with sender/time grouping. Threaded mode groups by `correlationId` into `<qhorus-thread>` components.

- [ ] **Step 1: Write failing tests**

```typescript
// chat-demo/src/main/webui/src/qhorus/composites/qhorus-channel-feed.test.ts
import { describe, it, expect, afterEach } from 'vitest';
import './qhorus-channel-feed.js';
import '../primitives/qhorus-message.js';
import '../primitives/qhorus-thread.js';
import '../primitives/qhorus-reaction-bar.js';
import type { QhorusMessage } from '../types.js';

function msg(id: string, overrides: Partial<QhorusMessage> = {}): QhorusMessage {
  return {
    id, channelId: 'ch-1', sender: 'agent-a', messageType: 'EVENT',
    actorType: 'AGENT', content: `Message ${id}`, replyCount: 0,
    artefactRefs: [], createdAt: '2026-07-07T12:00:00Z', ...overrides,
  };
}

afterEach(() => { document.body.innerHTML = ''; });

describe('qhorus-channel-feed', () => {
  it('renders messages in flat mode', async () => {
    const el = document.createElement('qhorus-channel-feed') as any;
    el.messages = [msg('1'), msg('2'), msg('3')];
    el.mode = 'flat';
    document.body.appendChild(el);
    await el.updateComplete;

    const msgs = el.shadowRoot!.querySelectorAll('qhorus-message');
    expect(msgs.length).toBe(3);
  });

  it('groups consecutive messages from same sender within 2 min', async () => {
    const el = document.createElement('qhorus-channel-feed') as any;
    el.messages = [
      msg('1', { sender: 'alice', createdAt: '2026-07-07T12:00:00Z' }),
      msg('2', { sender: 'alice', createdAt: '2026-07-07T12:00:30Z' }),
      msg('3', { sender: 'bob', createdAt: '2026-07-07T12:01:00Z' }),
    ];
    el.mode = 'flat';
    document.body.appendChild(el);
    await el.updateComplete;

    const headers = el.shadowRoot!.querySelectorAll('.message-group-header');
    expect(headers.length).toBe(2);
  });

  it('renders threads in threaded mode', async () => {
    const el = document.createElement('qhorus-channel-feed') as any;
    el.messages = [
      msg('1', { messageType: 'COMMAND', correlationId: 'corr-1' }),
      msg('2', { messageType: 'STATUS', correlationId: 'corr-1' }),
      msg('3', { messageType: 'DONE', correlationId: 'corr-1' }),
      msg('4', { messageType: 'EVENT' }),
    ];
    el.mode = 'threaded';
    document.body.appendChild(el);
    await el.updateComplete;

    const threads = el.shadowRoot!.querySelectorAll('qhorus-thread');
    expect(threads.length).toBe(1);
    const standalone = el.shadowRoot!.querySelectorAll(':scope > .message-item > qhorus-message');
    expect(standalone.length).toBeGreaterThanOrEqual(1);
  });

  it('defaults to flat mode', async () => {
    const el = document.createElement('qhorus-channel-feed') as any;
    el.messages = [msg('1')];
    document.body.appendChild(el);
    await el.updateComplete;

    expect(el.mode).toBe('flat');
  });

  it('renders mode toggle buttons', async () => {
    const el = document.createElement('qhorus-channel-feed') as any;
    el.messages = [msg('1')];
    document.body.appendChild(el);
    await el.updateComplete;

    const buttons = el.shadowRoot!.querySelectorAll('.mode-toggle button');
    expect(buttons.length).toBe(2);
    expect(buttons[0].textContent!.trim()).toBe('Flat');
    expect(buttons[1].textContent!.trim()).toBe('Threaded');
  });

  it('passes commitments to threads', async () => {
    const el = document.createElement('qhorus-channel-feed') as any;
    el.messages = [
      msg('1', { messageType: 'COMMAND', correlationId: 'c1', commitmentId: 'cm1' }),
      msg('2', { messageType: 'DONE', correlationId: 'c1' }),
    ];
    el.commitments = new Map([['cm1', 'FULFILLED']]);
    el.mode = 'threaded';
    document.body.appendChild(el);
    await el.updateComplete;

    const thread = el.shadowRoot!.querySelector('qhorus-thread') as any;
    expect(thread.commitmentState).toBe('FULFILLED');
  });
});
```

- [ ] **Step 2: Implement qhorus-channel-feed.ts**

The implementation groups messages by correlationId for threaded mode, and by sender/time for flat mode grouping. Uses `LiveRegionMixin` for screen reader announcements on new messages and `RovingTabindexMixin` for keyboard navigation.

```typescript
// chat-demo/src/main/webui/src/qhorus/composites/qhorus-channel-feed.ts
import { LitElement, html, css, nothing } from 'lit';
import { customElement, property, state } from 'lit/decorators.js';
import { LiveRegionMixin, KeyboardShortcutMixin } from '@casehubio/blocks-ui-core';
import type { QhorusMessage, Reaction, CommitmentState } from '../types.js';
import { emitChatEvent, ChatEventTopics } from '../events.js';
import '../primitives/qhorus-message.js';
import '../primitives/qhorus-thread.js';

interface MessageGroup {
  sender: string;
  actorType: string;
  messages: QhorusMessage[];
}

interface ThreadGroup {
  correlationId: string;
  root: QhorusMessage;
  replies: QhorusMessage[];
  commitmentState?: CommitmentState;
}

const QhorusChannelFeedBase = LiveRegionMixin(KeyboardShortcutMixin(LitElement));

@customElement('qhorus-channel-feed')
export class QhorusChannelFeedElement extends QhorusChannelFeedBase {
  @property({ type: Array }) messages: QhorusMessage[] = [];
  @property({ type: Array }) reactions: Reaction[] = [];
  @property({ type: Object }) commitments: Map<string, CommitmentState> = new Map();
  @property({ type: String }) mode: 'flat' | 'threaded' = 'flat';

  @state() private _prevMessageCount = 0;

  static override readonly styles = css`
    :host {
      display: flex;
      flex-direction: column;
      height: 100%;
      overflow: hidden;
    }
    .toolbar {
      display: flex;
      align-items: center;
      gap: var(--pages-space-2, 8px);
      padding: var(--pages-space-2, 8px) var(--pages-space-4, 16px);
      border-bottom: 1px solid var(--pages-neutral-4, #e5e5e5);
      flex-shrink: 0;
    }
    .mode-toggle {
      display: flex;
      gap: 0;
      border: 1px solid var(--pages-neutral-5, #d4d4d4);
      border-radius: var(--pages-radius-md, 6px);
      overflow: hidden;
    }
    .mode-toggle button {
      padding: var(--pages-space-1, 4px) var(--pages-space-3, 12px);
      font-size: var(--pages-font-size-xs, 11px);
      border: none;
      background: var(--pages-neutral-1, #fafafa);
      cursor: pointer;
      color: var(--pages-neutral-9, #737373);
    }
    .mode-toggle button.active {
      background: var(--pages-accent-3, #e0e7ff);
      color: var(--pages-accent-11, #3730a3);
      font-weight: var(--pages-font-weight-medium, 500);
    }
    .mode-toggle button + button {
      border-left: 1px solid var(--pages-neutral-5, #d4d4d4);
    }
    .feed {
      flex: 1;
      overflow-y: auto;
      scroll-behavior: smooth;
    }
    @media (prefers-reduced-motion: reduce) {
      .feed { scroll-behavior: auto; }
    }
    .message-group-header {
      display: flex;
      align-items: center;
      gap: var(--pages-space-2, 8px);
      padding: var(--pages-space-3, 12px) var(--pages-space-4, 16px) 0;
    }
    .group-sender {
      font-weight: var(--pages-font-weight-semibold, 600);
      font-size: var(--pages-font-size-sm, 13px);
    }
    .message-item { }
    .empty {
      display: flex;
      align-items: center;
      justify-content: center;
      height: 100%;
      color: var(--pages-neutral-8, #888);
      font-size: var(--pages-font-size-sm, 13px);
    }
  `;

  private _setMode(mode: 'flat' | 'threaded') {
    this.mode = mode;
  }

  private _reactionsFor(messageId: string): Reaction[] {
    return this.reactions.filter(r => r.messageId === messageId);
  }

  private _groupFlat(): MessageGroup[] {
    const groups: MessageGroup[] = [];
    const TWO_MINUTES = 2 * 60 * 1000;

    for (const msg of this.messages) {
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

  private _groupThreaded(): Array<ThreadGroup | QhorusMessage> {
    const threads = new Map<string, { root: QhorusMessage; replies: QhorusMessage[] }>();
    const standalone: QhorusMessage[] = [];

    for (const msg of this.messages) {
      if (msg.correlationId) {
        const existing = threads.get(msg.correlationId);
        if (existing) {
          existing.replies = [...existing.replies, msg];
        } else {
          threads.set(msg.correlationId, { root: msg, replies: [] });
        }
      } else {
        standalone.push(msg);
      }
    }

    const result: Array<ThreadGroup | QhorusMessage> = [];
    const threadArray = [...threads.values()];

    let tIdx = 0;
    let sIdx = 0;
    const allItems = [
      ...threadArray.map(t => ({ time: t.root.createdAt, type: 'thread' as const, data: t })),
      ...standalone.map(m => ({ time: m.createdAt, type: 'message' as const, data: m })),
    ].sort((a, b) => new Date(a.time).getTime() - new Date(b.time).getTime());

    for (const item of allItems) {
      if (item.type === 'thread') {
        const t = item.data as { root: QhorusMessage; replies: QhorusMessage[] };
        const commitmentState = t.root.commitmentId
          ? this.commitments.get(t.root.commitmentId)
          : undefined;
        result.push({ correlationId: t.root.correlationId!, ...t, commitmentState });
      } else {
        result.push(item.data as QhorusMessage);
      }
    }
    return result;
  }

  override updated(changed: Map<string, unknown>) {
    if (changed.has('messages') && this.messages.length > this._prevMessageCount) {
      this.announce(`New message from ${this.messages[this.messages.length - 1]?.sender}`);
      this._prevMessageCount = this.messages.length;
    }
  }

  private _selectMessage(msg: QhorusMessage) {
    emitChatEvent(this, ChatEventTopics.MESSAGE_SELECTED, { message: msg });
  }

  override render() {
    return html`
      <div class="toolbar">
        <div class="mode-toggle">
          <button class=${this.mode === 'flat' ? 'active' : ''}
                  @click=${() => this._setMode('flat')}>Flat</button>
          <button class=${this.mode === 'threaded' ? 'active' : ''}
                  @click=${() => this._setMode('threaded')}>Threaded</button>
        </div>
      </div>
      <div class="feed">
        ${this.messages.length === 0 ? html`
          <div class="empty">No messages yet</div>
        ` : this.mode === 'flat'
          ? this._renderFlat()
          : this._renderThreaded()
        }
      </div>
    `;
  }

  private _renderFlat() {
    return this._groupFlat().map(group => html`
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
        `)}
      </div>
    `);
  }

  private _renderThreaded() {
    return this._groupThreaded().map(item => {
      if ('correlationId' in item && 'root' in item) {
        const thread = item as ThreadGroup;
        return html`
          <qhorus-thread .rootMessage=${thread.root}
                         .replies=${thread.replies}
                         .commitmentState=${thread.commitmentState}>
          </qhorus-thread>
        `;
      }
      const msg = item as QhorusMessage;
      return html`
        <div class="message-item" @click=${() => this._selectMessage(msg)}>
          <qhorus-message .message=${msg}
                          .reactions=${this._reactionsFor(msg.id)}>
          </qhorus-message>
        </div>
      `;
    });
  }
}

declare global {
  interface HTMLElementTagNameMap {
    'qhorus-channel-feed': QhorusChannelFeedElement;
  }
}
```

- [ ] **Step 3: Run tests and commit**

```bash
cd chat-demo/src/main/webui && npx vitest run src/qhorus/composites/qhorus-channel-feed.test.ts
git add chat-demo/src/main/webui/src/qhorus/composites/qhorus-channel-feed.ts chat-demo/src/main/webui/src/qhorus/composites/qhorus-channel-feed.test.ts
git commit -m "feat(chat-demo): channel-feed composite — flat/threaded modes, sender grouping, a11y"
```

---

### Task 6: `<qhorus-channel-nav>` Composite

**Files:**
- Create: `chat-demo/src/main/webui/src/qhorus/composites/qhorus-channel-nav.ts`
- Create: `chat-demo/src/main/webui/src/qhorus/composites/qhorus-channel-nav.test.ts`

**Interfaces:**
- Consumes: `QhorusChannel`, `emitChatEvent`, `ChatEventTopics` from Task 1
- Produces: `<qhorus-channel-nav>` — Properties: `channels: QhorusChannel[]`, `selectedChannelId?: string`. Emits: `chat:select-channel`, `chat:create-channel`, `chat:delete-channel`

This component uses `RovingTabindexMixin` for keyboard navigation and `FocusTrapMixin` for modal dialogs (create/delete channel). Channel list items are clickable; selected channel is highlighted. Create/delete use simple prompt/confirm for Phase 1 (blocks-ui `<blocks-confirm-dialog>` can be adopted in Phase 2).

- [ ] **Step 1: Write tests, implement, run, commit**

Follow the same TDD pattern as Tasks 2–5. Tests cover:
- Renders list of channels
- Highlights selected channel
- Emits `chat:select-channel` on click
- Renders create channel button
- Emits `chat:create-channel` with name
- Renders delete button per channel
- Emits `chat:delete-channel` with channelId
- Keyboard navigation (arrow keys, Enter to select)
- Shows channel semantic icon (APPEND, BARRIER, etc.)

```bash
git commit -m "feat(chat-demo): channel-nav composite — channel list, selection, create/delete"
```

---

### Task 7: `<qhorus-member-panel>` Composite

**Files:**
- Create: `chat-demo/src/main/webui/src/qhorus/composites/qhorus-member-panel.ts`
- Create: `chat-demo/src/main/webui/src/qhorus/composites/qhorus-member-panel.test.ts`

**Interfaces:**
- Consumes: `ChannelMember`, `PresenceState` from Task 1
- Produces: `<qhorus-member-panel>` — Properties: `members: ChannelMember[]`, `presence: PresenceState[]`

Members sorted by presence status (online → away → offline), then alphabetically. Presence dots color-coded. Actor type badges (human/agent/system). Role indicator for moderators.

- [ ] **Step 1: Write tests, implement, run, commit**

Tests cover:
- Renders member list sorted by presence
- Shows presence dot with correct color
- Shows actor type badge
- Shows moderator badge for moderators
- Groups by presence status with section headers
- Handles empty member list

```bash
git commit -m "feat(chat-demo): member-panel composite — sorted members, presence dots, actor badges"
```

---

### Task 8: Chat-Demo Adapter and Workbench Integration

**Files:**
- Create: `chat-demo/src/main/webui/src/qhorus/workbench/chat-demo-adapter.ts`
- Create: `chat-demo/src/main/webui/src/qhorus/workbench/chat-demo-adapter.test.ts`
- Create: `chat-demo/src/main/webui/src/qhorus/workbench/qhorus-workbench.ts`
- Create: `chat-demo/src/main/webui/src/qhorus/workbench/qhorus-workbench.test.ts`
- Create: `chat-demo/src/main/webui/src/qhorus/index.ts`

**Interfaces:**
- Consumes: All composites from Tasks 5–7, all types from Task 1, `authenticatedFetch`/`getToken` from `auth.ts`
- Produces:
  - `ChatDemoAdapter` — class that manages WebSocket connection, translates ws-data ops to typed arrays, handles REST calls. Methods: `connect(url, token)`, `disconnect()`, `sendMessage(payload)`, `createChannel(payload)`, `deleteChannel(channelId)`, `addReaction(messageId, emoji)`, `removeReaction(messageId, emoji)`. Properties: `channels`, `messages`, `members`, `presence`, `reactions`
  - `<qhorus-workbench>` — assembles composites with pages layout, owns the adapter

- [ ] **Step 1: Write failing adapter tests**

```typescript
// chat-demo/src/main/webui/src/qhorus/workbench/chat-demo-adapter.test.ts
import { describe, it, expect, vi, beforeEach } from 'vitest';
import { ChatDemoAdapter } from './chat-demo-adapter.js';

describe('ChatDemoAdapter', () => {
  it('maps channel snapshot to QhorusChannel array', () => {
    const adapter = new ChatDemoAdapter();
    adapter.applyOp({
      op: 'snapshot', dataset: 'channels',
      rows: [['ch-1', 'general', 'General chat'], ['ch-2', 'random', 'Random']],
    });
    expect(adapter.channels.length).toBe(2);
    expect(adapter.channels[0].name).toBe('general');
    expect(adapter.channels[0].semantic).toBe('APPEND');
  });

  it('maps message snapshot with default messageType and actorType', () => {
    const adapter = new ChatDemoAdapter();
    adapter.applyOp({
      op: 'snapshot', dataset: 'messages',
      rows: [['ch-1', 'msg-1', null, 'alice', 'Hello', '2026-07-07T12:00:00Z']],
    });
    expect(adapter.messages.length).toBe(1);
    expect(adapter.messages[0].messageType).toBe('EVENT');
    expect(adapter.messages[0].actorType).toBe('HUMAN');
    expect(adapter.messages[0].content).toBe('Hello');
  });

  it('handles append op by adding to existing array', () => {
    const adapter = new ChatDemoAdapter();
    adapter.applyOp({
      op: 'snapshot', dataset: 'messages',
      rows: [['ch-1', 'msg-1', null, 'alice', 'First', '2026-07-07T12:00:00Z']],
    });
    adapter.applyOp({
      op: 'append', dataset: 'messages',
      row: ['ch-1', 'msg-2', null, 'bob', 'Second', '2026-07-07T12:01:00Z'],
    });
    expect(adapter.messages.length).toBe(2);
  });

  it('handles remove op', () => {
    const adapter = new ChatDemoAdapter();
    adapter.applyOp({
      op: 'snapshot', dataset: 'channels',
      rows: [['ch-1', 'general', ''], ['ch-2', 'random', '']],
    });
    adapter.applyOp({
      op: 'remove', dataset: 'channels', key: 'ch-2',
    });
    expect(adapter.channels.length).toBe(1);
    expect(adapter.channels[0].id).toBe('ch-1');
  });

  it('maps presence snapshot', () => {
    const adapter = new ChatDemoAdapter();
    adapter.applyOp({
      op: 'snapshot', dataset: 'presence',
      rows: [['alice', 'ONLINE'], ['bob', 'AWAY']],
    });
    expect(adapter.presence.length).toBe(2);
    expect(adapter.presence[0].status).toBe('ONLINE');
    expect(adapter.presence[1].status).toBe('AWAY');
  });

  it('maps member snapshot', () => {
    const adapter = new ChatDemoAdapter();
    adapter.applyOp({
      op: 'snapshot', dataset: 'members',
      rows: [['m-1', 'ch-1', 'alice', 'Alice']],
    });
    expect(adapter.members.length).toBe(1);
    expect(adapter.members[0].displayName).toBe('Alice');
    expect(adapter.members[0].role).toBe('PARTICIPANT');
  });

  it('notifies listeners on data change', () => {
    const adapter = new ChatDemoAdapter();
    const listener = vi.fn();
    adapter.onChange(listener);

    adapter.applyOp({
      op: 'snapshot', dataset: 'channels',
      rows: [['ch-1', 'general', '']],
    });

    expect(listener).toHaveBeenCalledWith('channels');
  });
});
```

- [ ] **Step 2: Implement ChatDemoAdapter**

```typescript
// chat-demo/src/main/webui/src/qhorus/workbench/chat-demo-adapter.ts
import type {
  QhorusMessage, QhorusChannel, Reaction, ChannelMember, PresenceState,
} from '../types.js';

interface WsOp {
  op: 'snapshot' | 'append' | 'replace' | 'remove';
  dataset: string;
  rows?: unknown[][];
  row?: unknown[];
  key?: string;
}

type ChangeListener = (dataset: string) => void;

export class ChatDemoAdapter {
  channels: QhorusChannel[] = [];
  messages: QhorusMessage[] = [];
  reactions: Reaction[] = [];
  members: ChannelMember[] = [];
  presence: PresenceState[] = [];

  private _listeners: ChangeListener[] = [];

  onChange(fn: ChangeListener) { this._listeners.push(fn); }
  offChange(fn: ChangeListener) { this._listeners = this._listeners.filter(l => l !== fn); }
  private _notify(dataset: string) { for (const fn of this._listeners) fn(dataset); }

  applyOp(op: WsOp) {
    switch (op.dataset) {
      case 'channels': this._applyChannels(op); break;
      case 'messages': this._applyMessages(op); break;
      case 'reactions': this._applyReactions(op); break;
      case 'members': this._applyMembers(op); break;
      case 'presence': this._applyPresence(op); break;
    }
    this._notify(op.dataset);
  }

  private _applyChannels(op: WsOp) {
    if (op.op === 'snapshot') {
      this.channels = (op.rows ?? []).map(r => this._toChannel(r));
    } else if (op.op === 'append' && op.row) {
      this.channels = [...this.channels, this._toChannel(op.row)];
    } else if (op.op === 'remove' && op.key) {
      this.channels = this.channels.filter(c => c.id !== op.key);
    }
  }

  private _toChannel(row: unknown[]): QhorusChannel {
    return {
      id: row[0] as string,
      name: row[1] as string,
      description: (row[2] as string) || undefined,
      semantic: 'APPEND',
      paused: false,
    };
  }

  private _applyMessages(op: WsOp) {
    if (op.op === 'snapshot') {
      this.messages = (op.rows ?? []).map(r => this._toMessage(r));
    } else if (op.op === 'append' && op.row) {
      this.messages = [...this.messages, this._toMessage(op.row)];
    } else if (op.op === 'remove' && op.key) {
      this.messages = this.messages.filter(m => m.id !== op.key);
    }
  }

  private _toMessage(row: unknown[]): QhorusMessage {
    return {
      id: row[1] as string,
      channelId: row[0] as string,
      sender: row[3] as string,
      messageType: 'EVENT',
      actorType: 'HUMAN',
      content: row[4] as string,
      correlationId: undefined,
      inReplyTo: (row[2] as string) || undefined,
      replyCount: 0,
      artefactRefs: [],
      createdAt: row[5] as string,
    };
  }

  private _applyReactions(op: WsOp) {
    if (op.op === 'snapshot') {
      this.reactions = (op.rows ?? []).map(r => ({
        messageId: r[0] as string, emoji: r[1] as string,
        actorId: '', createdAt: '',
      }));
    } else if (op.op === 'append' && op.row) {
      this.reactions = [...this.reactions, {
        messageId: op.row[0] as string, emoji: op.row[1] as string,
        actorId: '', createdAt: '',
      }];
    }
  }

  private _applyMembers(op: WsOp) {
    if (op.op === 'snapshot') {
      this.members = (op.rows ?? []).map(r => ({
        channelId: r[1] as string, memberId: r[2] as string,
        displayName: r[3] as string, role: 'PARTICIPANT' as const,
      }));
    } else if (op.op === 'append' && op.row) {
      this.members = [...this.members, {
        channelId: op.row[1] as string, memberId: op.row[2] as string,
        displayName: op.row[3] as string, role: 'PARTICIPANT' as const,
      }];
    }
  }

  private _applyPresence(op: WsOp) {
    if (op.op === 'snapshot') {
      this.presence = (op.rows ?? []).map(r => ({
        memberId: r[0] as string,
        status: r[1] as PresenceState['status'],
      }));
    } else if (op.op === 'replace' && op.row) {
      this.presence = this.presence.map(p =>
        p.memberId === op.row![0] ? { ...p, status: op.row![1] as PresenceState['status'] } : p
      );
    }
  }
}
```

- [ ] **Step 3: Run adapter tests**

```bash
cd chat-demo/src/main/webui && npx vitest run src/qhorus/workbench/chat-demo-adapter.test.ts
```

- [ ] **Step 4: Implement qhorus-workbench.ts**

The workbench:
1. Creates the `ChatDemoAdapter` and connects WebSocket
2. Listens for `chat:*` events from composites and translates to REST calls
3. Assembles the layout using pages `split()`, `dockBar()`, `hostPanel()` primitives
4. Pushes adapter data to composites via `@property()` setters on data change

```typescript
// chat-demo/src/main/webui/src/qhorus/workbench/qhorus-workbench.ts
import { LitElement, html, css } from 'lit';
import { customElement, property, state } from 'lit/decorators.js';
import { ChatDemoAdapter } from './chat-demo-adapter.js';
import { ChatEventTopics } from '../events.js';
import type { SendMessagePayload, ReactPayload, CreateChannelPayload } from '../events.js';
import type { QhorusMessage, QhorusChannel, Reaction, ChannelMember, PresenceState, CommitmentState } from '../types.js';
import { getToken, getIdentity, authenticatedFetch } from '../../auth.js';
import '../composites/qhorus-channel-feed.js';
import '../composites/qhorus-channel-nav.js';
import '../composites/qhorus-member-panel.js';
import '../primitives/qhorus-message-input.js';

@customElement('qhorus-workbench')
export class QhorusWorkbenchElement extends LitElement {
  @property({ type: String }) endpoint = '';
  @property({ type: String }) restBase = '/api';

  @state() private _channels: QhorusChannel[] = [];
  @state() private _messages: QhorusMessage[] = [];
  @state() private _reactions: Reaction[] = [];
  @state() private _members: ChannelMember[] = [];
  @state() private _presence: PresenceState[] = [];
  @state() private _selectedChannelId = '';
  @state() private _replyTo?: { messageId: string; senderName: string };

  private _adapter = new ChatDemoAdapter();
  private _ws?: WebSocket;

  static override readonly styles = css`
    :host {
      display: flex;
      height: 100%;
      overflow: hidden;
      font-family: var(--pages-font-family, 'Inter', system-ui, sans-serif);
    }
    .nav-panel {
      width: 240px;
      flex-shrink: 0;
      border-right: 1px solid var(--pages-neutral-4, #e5e5e5);
      overflow-y: auto;
    }
    .main-panel {
      flex: 1;
      display: flex;
      flex-direction: column;
      min-width: 0;
    }
    .member-panel {
      width: 220px;
      flex-shrink: 0;
      border-left: 1px solid var(--pages-neutral-4, #e5e5e5);
      overflow-y: auto;
    }
  `;

  override connectedCallback() {
    super.connectedCallback();
    this._adapter.onChange(this._onDataChange);
    this.addEventListener('pages-event', this._onChatEvent as EventListener);
    this._connect();
  }

  override disconnectedCallback() {
    super.disconnectedCallback();
    this._adapter.offChange(this._onDataChange);
    this.removeEventListener('pages-event', this._onChatEvent as EventListener);
    this._ws?.close();
  }

  private _connect() {
    const token = getToken();
    if (!token || !this.endpoint) return;

    const proto = location.protocol === 'https:' ? 'wss:' : 'ws:';
    const url = `${proto}//${location.host}${this.endpoint}?token=${token}`;
    this._ws = new WebSocket(url);

    this._ws.onmessage = (e) => {
      try {
        const op = JSON.parse(e.data);
        this._adapter.applyOp(op);
      } catch { /* malformed message */ }
    };

    this._ws.onclose = () => {
      setTimeout(() => this._connect(), 3000);
    };
  }

  private _onDataChange = (dataset: string) => {
    this._channels = this._adapter.channels;
    this._messages = this._adapter.messages;
    this._reactions = this._adapter.reactions;
    this._members = this._adapter.members;
    this._presence = this._adapter.presence;
  };

  private _onChatEvent = (e: CustomEvent) => {
    const { topic, payload } = e.detail;

    switch (topic) {
      case ChatEventTopics.SELECT_CHANNEL:
        this._selectedChannelId = (payload as { channelId: string }).channelId;
        break;
      case ChatEventTopics.SEND_MESSAGE:
        this._sendMessage(payload as SendMessagePayload);
        break;
      case ChatEventTopics.CREATE_CHANNEL:
        this._createChannel(payload as CreateChannelPayload);
        break;
      case ChatEventTopics.DELETE_CHANNEL:
        this._deleteChannel((payload as { channelId: string }).channelId);
        break;
      case ChatEventTopics.REACT:
        this._addReaction(payload as ReactPayload);
        break;
      case ChatEventTopics.UNREACT:
        this._removeReaction(payload as ReactPayload);
        break;
      case ChatEventTopics.MESSAGE_SELECTED:
        this._replyTo = {
          messageId: (payload as { message: QhorusMessage }).message.id,
          senderName: (payload as { message: QhorusMessage }).message.sender,
        };
        break;
    }
  };

  private async _sendMessage(payload: SendMessagePayload) {
    const url = payload.inReplyTo
      ? `${this.restBase}/channels/${payload.channelId}/messages/${payload.inReplyTo}/replies`
      : `${this.restBase}/channels/${payload.channelId}/messages`;
    await authenticatedFetch(url, {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({ text: payload.content }),
    });
    this._replyTo = undefined;
  }

  private async _createChannel(payload: CreateChannelPayload) {
    await authenticatedFetch(`${this.restBase}/channels`, {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({ name: payload.name }),
    });
  }

  private async _deleteChannel(channelId: string) {
    await authenticatedFetch(`${this.restBase}/channels/${channelId}`, { method: 'DELETE' });
  }

  private async _addReaction(payload: ReactPayload) {
    const msg = this._messages.find(m => m.id === payload.messageId);
    if (!msg) return;
    await authenticatedFetch(
      `${this.restBase}/channels/${msg.channelId}/messages/${payload.messageId}/reactions`,
      { method: 'POST', headers: { 'Content-Type': 'application/json' }, body: JSON.stringify({ emoji: payload.emoji }) },
    );
  }

  private async _removeReaction(payload: ReactPayload) {
    const msg = this._messages.find(m => m.id === payload.messageId);
    if (!msg) return;
    await authenticatedFetch(
      `${this.restBase}/channels/${msg.channelId}/messages/${payload.messageId}/reactions/${payload.emoji}`,
      { method: 'DELETE' },
    );
  }

  private _filteredMessages(): QhorusMessage[] {
    if (!this._selectedChannelId) return [];
    return this._messages.filter(m => m.channelId === this._selectedChannelId);
  }

  private _filteredMembers(): ChannelMember[] {
    if (!this._selectedChannelId) return [];
    return this._members.filter(m => m.channelId === this._selectedChannelId);
  }

  override render() {
    const identity = getIdentity();

    return html`
      <div class="nav-panel">
        <qhorus-channel-nav
          .channels=${this._channels}
          .selectedChannelId=${this._selectedChannelId}>
        </qhorus-channel-nav>
      </div>
      <div class="main-panel">
        <qhorus-channel-feed
          .messages=${this._filteredMessages()}
          .reactions=${this._reactions}>
        </qhorus-channel-feed>
        <qhorus-message-input
          .channelId=${this._selectedChannelId}
          .replyTo=${this._replyTo}>
        </qhorus-message-input>
      </div>
      <div class="member-panel">
        <qhorus-member-panel
          .members=${this._filteredMembers()}
          .presence=${this._presence}>
        </qhorus-member-panel>
      </div>
    `;
  }
}

declare global {
  interface HTMLElementTagNameMap {
    'qhorus-workbench': QhorusWorkbenchElement;
  }
}
```

- [ ] **Step 5: Write workbench tests**

Test that the workbench:
- Renders all three panels (nav, feed, members)
- Routes `chat:select-channel` to update feed filter
- Routes `chat:send-message` to REST call
- Routes `chat:create-channel` and `chat:delete-channel` to REST
- Passes filtered messages to feed (only selected channel)
- Passes filtered members to member panel

- [ ] **Step 6: Create new entry point**

```typescript
// chat-demo/src/main/webui/src/qhorus/index.ts
import { loadSite, registerPanel } from '@casehubio/pages-runtime';
import { columns, split, dockBar, hostPanel, withId } from '@casehubio/pages-ui';
import { PagesDevAuth, PagesIdentity } from '@casehubio/pages-ui';
import './workbench/qhorus-workbench.js';

registerPanel('chat-workbench', 'qhorus-workbench');

const identityBar = columns([1, 0], [hostPanel('identity')], []);
registerPanel('identity', 'pages-identity');

const app = columns([0, 1],
  [withId('identity-bar', identityBar)],
  [withId('chat', hostPanel('chat-workbench', {
    endpoint: '/ws/chat',
    restBase: '/api',
  }))],
);

document.addEventListener('pages-auth-success', () => {
  const container = document.getElementById('app');
  if (!container) return;
  loadSite(container, app).then(site => site.setTheme('dark'));
});
```

- [ ] **Step 7: Run all tests and commit**

```bash
cd chat-demo/src/main/webui && npx vitest run src/qhorus/
git add chat-demo/src/main/webui/src/qhorus/workbench/ chat-demo/src/main/webui/src/qhorus/index.ts
git commit -m "feat(chat-demo): workbench + chat-demo adapter — WebSocket data flow, REST event routing"
```

---

### Task 9: Swap — Replace Old UI and Delete Old Code

**Files:**
- Modify: `chat-demo/src/main/webui/src/index.ts` → replace with import of `qhorus/index.ts`
- Delete: `chat-demo/src/main/webui/src/panels/channel-sidebar.ts`
- Delete: `chat-demo/src/main/webui/src/panels/message-list.ts`
- Delete: `chat-demo/src/main/webui/src/panels/message-input.ts`
- Delete: `chat-demo/src/main/webui/src/panels/member-list.ts`
- Delete: `chat-demo/src/main/webui/src/responsive.ts`
- Delete: `chat-demo/src/main/webui/src/responsive.test.ts`
- Delete: `chat-demo/src/main/webui/src/layout-fit.test.ts`
- Delete: `chat-demo/src/main/webui/src/test-helpers.ts`
- Modify: `chat-demo/src/main/webui/esbuild.config.mjs` → update entry point

**Interfaces:**
- Consumes: All previous tasks
- Produces: Working chat-demo UI powered by the new qhorus components

- [ ] **Step 1: Replace index.ts entry point**

Replace `chat-demo/src/main/webui/src/index.ts` content with:

```typescript
// Re-export from qhorus entry point
export * from './qhorus/index.js';
```

- [ ] **Step 2: Update esbuild.config.mjs**

The entry point stays the same (`src/index.ts`) — it now re-exports from the qhorus module. No esbuild config change needed if the entry point hasn't changed.

- [ ] **Step 3: Delete old files**

```bash
rm chat-demo/src/main/webui/src/panels/channel-sidebar.ts
rm chat-demo/src/main/webui/src/panels/message-list.ts
rm chat-demo/src/main/webui/src/panels/message-input.ts
rm chat-demo/src/main/webui/src/panels/member-list.ts
rm chat-demo/src/main/webui/src/responsive.ts
rm chat-demo/src/main/webui/src/responsive.test.ts
rm chat-demo/src/main/webui/src/layout-fit.test.ts
rm chat-demo/src/main/webui/src/test-helpers.ts
rmdir chat-demo/src/main/webui/src/panels
```

- [ ] **Step 4: Run full test suite**

```bash
cd chat-demo/src/main/webui && npx vitest run
```

Expected: all qhorus tests pass, old tests are gone, auth tests still pass.

- [ ] **Step 5: Build and verify**

```bash
cd chat-demo/src/main/webui && npm run build
```

Expected: builds successfully with no errors.

- [ ] **Step 6: Manual verification**

Start the chat-demo backend and verify in a browser:
```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn -f chat-demo/pom.xml quarkus:dev -Pdemo -Pui
```

Verify:
- Login page works (pages-dev-auth)
- Channel list renders
- Messages render with speech act badges (if qhorus backend) or plain text (chat-demo backend)
- Sending messages works
- Reactions render
- Member panel shows presence
- Flat/Threaded toggle works
- Responsive layout via pages split panels

- [ ] **Step 7: Commit the swap**

```bash
git add -A chat-demo/src/main/webui/
git commit -m "feat(chat-demo): swap to qhorus UI — delete old HTMLElement components, Lit UI live"
```

---

## Deferred Items → GitHub Issues

Before completing Phase 1, file these issues for tracking:

| Issue title | Phase | Description |
|------------|-------|-------------|
| feat(chat-demo): dockable contextual panels (artifact, task, correlation) | 2 | Implement `qhorus-artifact-panel`, `qhorus-task-panel`, `qhorus-correlation-panel` with dockBar integration |
| feat(chat-demo): progressive disclosure on message expand | 2 | Click-to-expand showing full correlation chain, artefact details, commitment details |
| feat(chat-demo): emoji reaction palette | 3 | Full emoji picker component for add-reaction. Depends on qhorus#328 (Reactions) |
| feat(chat-demo): rich artefact references with selection scope | 3 | Clickable artefact chips with type icons, selection scope preview. Depends on qhorus#328 |
| feat(chat-demo): topic navigator and topic view mode | 4 | Topic bar, topic filtering, topic resolve/rename. Depends on qhorus#328 (Topic field on Message) |
| feat(chat-demo): space-based channel hierarchy | 4 | Tree navigation with normative triple grouping. Depends on qhorus#328 (Space) |
| feat(chat-demo): channel membership and presence model | 4 | Distinct from ACL. Depends on qhorus#328 (ChannelMembership, Presence) |
| feat(claudony): embed qhorus workbench for live channel observation | 5 | Wire workbench to qhorus SSE, normative triple view, human participation |

---

## Dependency Graph

```
Task 1 (Scaffold + Types)
  ↓
Task 2 (qhorus-message)
  ↓
Task 3 (reaction-bar + thread)    Task 4 (message-input)
  ↓                                   ↓
Task 5 (channel-feed)    Task 6 (channel-nav)    Task 7 (member-panel)
  ↓                           ↓                        ↓
  └─────────── Task 8 (adapter + workbench) ───────────┘
                              ↓
                    Task 9 (swap + cleanup)
```

Tasks 4, 6, 7 can run in parallel once Task 1 is complete.
Tasks 5 depends on Tasks 2 and 3.
Task 8 depends on Tasks 5, 6, and 7.
Task 9 depends on Task 8.
