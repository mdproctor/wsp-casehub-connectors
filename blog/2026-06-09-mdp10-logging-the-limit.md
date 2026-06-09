---
layout: post
title: "Logging the limit"
date: 2026-06-09
type: phase-update
entry_type: note
subtype: diary
projects: [casehub-connectors]
tags: [slack, testing, java]
---

The 200-channel cap in `conversations.list` has been there since we shipped
`listChannels()`. An operator with a large workspace would get a partial channel
list with no indication anything was missing — the response silently ends at 200.

The fix is a single `containsKey` check and a `LOG.warning`. The interesting part
was the test.

Asserting that a method emits a warning isn't as natural as asserting it returns
the right value. My first instinct was to reach for logback-test or Mockito spying
on the logger. But this is a pure unit test — no `@QuarkusTest`, no Quarkus at all
— so the JUL logger is running unmodified. You can call `Logger.addHandler()`,
install a temporary handler that collects `LogRecord`s, wrap the action in a
`try/finally`, and assert on the list. Zero extra dependencies.

Three cases felt right: response with a non-empty cursor (should warn), response
with no `response_metadata` at all (should not), and response with `response_metadata`
present but no `next_cursor` key (should not). The third case surfaced in code review
— Claude flagged that I'd covered the obvious cases and missed the subtler shape.
`getString("next_cursor", "")` handles it correctly; the gap was in the tests, not
the implementation.

The JUL-in-Quarkus variant of this pattern is a separate trap: inside `@QuarkusTest`,
JBoss LogManager's `Level.WARN` is not `java.util.logging.Level.WARNING` — object
identity comparison fails silently. That one's already in the garden from a previous
session. The add-a-handler technique itself is now there too.
