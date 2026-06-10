---
layout: post
title: "Three decisions inside a cursor loop"
date: 2026-06-10
type: phase-update
entry_type: note
subtype: diary
projects: [casehub-connectors]
tags: [slack, pagination, java]
---

`SlackBotClient.listChannels()` used to return the first 200 channels and log a WARNING when `next_cursor` was non-empty. Now it follows cursor pagination until exhausted. The loop itself is four lines; the design decisions inside it take more explaining.

## What to return when the loop fails mid-way

Empty list is the obvious answer. It's wrong. An operator seeing an empty channel list can't tell whether the workspace has no channels or whether the third of ten requests returned `ratelimited`. Return the channels already accumulated, with a WARNING that includes the count and the error string — callers know exactly what happened.

This forced a structural change. The old `parseChannels()` returned `List<DiscoveredTarget>` and discarded the cursor. A mid-loop failure needs both the accumulated channels and the reason. The replacement `parsePage()` returns a four-field record:

```java
private record PageResult(boolean ok, List<DiscoveredTarget> channels,
                          String nextCursor, String error) {}
```

Each field exists because something breaks without it. Without `ok`, a `{"ok":false,"error":"ratelimited"}` response produces `PageResult(List.of(), "")` — indistinguishable from a normal last page; the loop thinks pagination is complete. Without `error`, the WARNING has nothing informative to say. And repurposing `nextCursor` to hold the error string when `ok` is false would make the field lie about its purpose. A dedicated field is the price of not lying.

## How to tell the page cap from clean completion

`MAX_PAGES = 50` means the loop exits either when `cursor` goes blank (last page reached normally via `break`) or when `pageNum` hits 50 with a cursor still pending. After the loop, `cursor.isBlank()` distinguishes them: blank means the break fired, non-blank means the cap fired. No boolean flag needed — the invariant already lives in `cursor`.

## The Jakarta JSON null trap

`JsonObject.getJsonObject("response_metadata")` returns `null` when the key is absent. Unlike `getString(name, "")` or `getBoolean(name, false)`, there's no overload that takes a default. Slack omits `response_metadata` entirely for workspaces with fewer than 200 channels, which makes a `containsKey` guard mandatory before accessing the nested object. Without it, every workspace under the limit throws a NPE from inside the parsing block — silent wrong-diagnosis territory.

Cursors also need URL-encoding, even though Slack uses URL-safe base64. Standard base64 `+` is technically valid in a URI query string per RFC 3986, so `URI.create()` won't throw — but some HTTP servers decode query parameters as `application/x-www-form-urlencoded`, where `+` means space. `URLEncoder.encode(cursor, UTF_8)` is unconditional.

The four-field record and the fail-soft return ended up formalised as a standing project protocol: any paginating HTTP client method in this codebase returns partial results with a WARNING rather than empty list on failure. We didn't plan that going in; it fell out of getting the design right.
