---
layout: post
title: "The Demo Gets a Face"
date: 2026-07-01
type: phase-update
entry_type: note
subtype: diary
projects: [casehub-connectors]
tags: [chat-demo, casehub-pages, quinoa, webui]
---

## The Demo Gets a Face

The chat-demo module has had a complete backend since the original #26 work — REST endpoints, WebSocket broadcasting, SQLite with a seed database. What it didn't have was anything to look at. Opening `localhost:8090` gave you a blank page and an invitation to `curl`.

This branch adds a casehub-pages frontend via Quinoa. Three-column chat workspace: channel sidebar on the left, message list in the centre, member panel with presence dots on the right. Both side panels dock — toggle them with icons and the message area fills the freed space. Dark mode by default, because every chat app worth its salt has dark mode.

## The Wire Protocol Mismatch

The first surprise was the wire protocol. The existing `ChatWebSocketBroadcaster` was already sending structured messages — dataset name, rows, operation type — but it used `type` instead of `op`, didn't send column metadata, and happily serialised Java booleans as JSON `true`/`false`. casehub-pages wants `op`, requires `columns` with `id`, `name`, and `type` on every operation, expects a monotonic `seq` counter, and demands that every row value be a string or null.

I also hadn't accounted for presence. The original broadcaster snapshot sent three datasets — channels, messages, members — but the member panel needs a fourth: a presence snapshot that tells the UI who's online the moment the connection opens. Without it, every member shows as grey until the first status change arrives. The presence snapshot required deduplication too — a member in three channels shouldn't produce three presence rows.

## Four Panels, One WebSocket

The architectural choice that shaped everything: a single WebSocket connection in the entry point, relaying messages as DOM events. Each Web Component panel listens for what it cares about. The channel sidebar listens for `dataset: "channels"`, the message list for `dataset: "messages"`, the member panel for both `members` and `presence`. No panel knows about the WebSocket directly.

Channel selection works the same way — the sidebar dispatches a `channel-selected` event, and the message list, member panel, and compose input all filter locally. Switching channels is instant because the data is already in memory.

The message list was the most involved panel. Message grouping — consecutive messages from the same sender within two minutes collapse into a single block without repeating the sender name — required careful index arithmetic. Scroll anchoring was the other wrinkle: auto-scroll to bottom on new messages, but only if the user is already at the bottom. If they've scrolled up to read history, show a "New messages" pill instead. The first implementation leaked scroll event listeners on every render; the design review caught it and we switched to delegated listeners on the shadow root.

## Profile Gating with an Absent Dependency

The Quinoa integration surfaced a gotcha worth knowing. `quarkus.quinoa.just-build=true` sounds like it disables everything — just build, don't serve. It doesn't. Quinoa still downloads Node.js and runs `npm install`. The only reliable way to conditionally disable Quinoa is to keep the dependency out of the classpath entirely. The `quarkus-quinoa` dependency sits inside a `-Pui` Maven profile. Without `-Pui`, Quinoa doesn't exist — the properties in `application.properties` are logged as warnings and ignored.

The result: `mvn clean install -Pdemo` builds the backend without Node.js. `mvn clean install -Pdemo -Pui` builds everything. `mvn quarkus:dev -Pdemo -Pui` gives you hot-reloading on both sides.
