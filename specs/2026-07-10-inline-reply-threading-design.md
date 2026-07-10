# Inline Reply Threading — Design Spec

**Issue:** casehubio/connectors#79
**Date:** 2026-07-10
**Status:** Approved

## Problem

The Flat/Threaded toggle in channel-feed is non-functional. Threaded mode groups
by `correlationId`, which the chat-demo adapter hardcodes to `undefined`. Both
modes render identically. Meanwhile, reply data (`inReplyTo`) exists in the
backend but is never surfaced.

## Solution

Remove the toggle. Render messages chronologically with inline reply threading:
messages with `inReplyTo` are grouped under their parent as collapsible reply
chains using the existing `<qhorus-thread>` component.

## Rendering Pipeline

```
1. Separate roots from replies (by inReplyTo)
2. Order roots chronologically with sender grouping
3. After each root that has replies, render <qhorus-thread> (collapsed)
```

This pipeline is a composable building block. When qhorus#328 lands:
- Topics mode adds a filter/group-by-topic step BEFORE step 1
- Correlation chain grouping overlays on top — distinct from reply threading
- Nothing built here needs to be torn out

## Changes

### `qhorus-channel-feed.ts`

- Remove `mode` property, `_setMode()`, toggle UI, `_groupThreaded()`, `_renderThreaded()`
- Add `_separateRootsAndReplies()` — returns `{ roots: QhorusMessage[], repliesByParent: Map<string, QhorusMessage[]> }`
- Modify `_groupFlat()` to accept roots only (exclude replies)
- In render: after each root message, check `repliesByParent` and render `<qhorus-thread>` if replies exist

### `chat-demo-adapter.ts`

- After snapshot/append, compute `replyCount` on each message by counting how many messages reference it via `inReplyTo`

### `<qhorus-thread>` — no changes

Already handles root + replies + collapse/expand + commitment state badge.

## Forward Compatibility (qhorus#328)

| Future feature | How it layers | Impact on this work |
|---------------|--------------|-------------------|
| Topics mode | Filter by `topic` before root/reply separation | None — step 1 of pipeline |
| Correlation chains | Group by `correlationId`, render as thread groups | None — separate concept from `inReplyTo` |
| Spaces | Channel container, no feed impact | None |

## Testing

- Existing thread primitive tests — unchanged
- Channel-feed tests — update to remove toggle assertions, add inline reply grouping tests
- Adapter tests — add replyCount computation tests
- Playwright — verify reply chains render inline, expand/collapse works
