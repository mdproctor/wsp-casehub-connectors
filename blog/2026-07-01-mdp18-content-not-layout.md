---
layout: post
title: "Content, not layout"
date: 2026-07-01
type: phase-update
entry_type: note
subtype: diary
projects: [casehub-connectors]
tags: [chat-spi, rich-content, slack, discord, mcp, design]
series: issue-37-rich-content-model
---

Discord embeds, Slack Block Kit, Teams Adaptive Cards, Google Chat Cards V2 — four platforms, four incompatible JSON schemas for saying "here's a title, some fields, and an image." The question for connectors was whether to pick one schema to rule them all or find the abstraction underneath.

## The research that changed the framing

I started with two options: a card model (flat, maps to Discord embeds) or a block model (compositional, maps to Slack/Teams). The block model looked architecturally superior — the industry has converged on it. Microsoft's Bot Framework, Matrix's MSC1767 extensible events, Google Chat Cards V2 all use ordered lists of typed elements.

But the consumers of this model are LLM agents calling MCP tools, and programmatic callers using the ChatPlatform SPI. Neither thinks in layout primitives. An LLM doesn't want to specify "put a divider here, then a context block with two mrkdwn elements." It wants to say "title, description, three fields, green."

That's the distinction: **content** (what to show) vs **layout** (how to arrange it). A divider between two sections is a layout decision. A title with fields is content. The abstraction should model content; each platform renderer decides the layout.

## RichCard — the content card

The model is a `RichCard` record with nine fields: title, description, url, color, fields (name/value/inline triples), thumbnailUrl, imageUrl, footer, author. A Builder because nine positional parameters with seven nullable is hostile to callers. `ChatContent` gains a `List<RichCard> cards` field alongside the existing text/markdown/attachments.

The translation to each platform is straightforward. Discord: `RichCard` → `DiscordEmbed`, near 1:1. Slack: `RichCard` → Block Kit JSON — header block for title, section for description, section with fields array, image block, context block for footer/author. Color drops silently on Slack (Block Kit doesn't support it). IRC ignores cards entirely and sends text.

The interesting constraint: `text` stays required even when cards are present. Discord supports embed-only messages — visually cleaner, no text above the card. We chose cross-platform consistency over Discord-specific polish. On Slack, `text` is the push notification fallback when blocks are rendered in-app. Every platform can always render text; not every platform can render cards.

## The tool consolidation

`send_slack_bot` and `send_discord` were both bypassing the ChatPlatform SPI entirely — talking directly to vendor HTTP clients. Two tools doing the same thing through different code paths. We replaced both with a single `send_chat(platform, channel, text, ...)` that routes through `ChatPlatformService`. Same for discovery: `list_discord_channels` became `list_chat_channels(platform)` with rich detail — topic, description, private flag, member count.

The `Channel` record gained `memberCount` (nullable `Integer`). Discord provides it via the guild endpoint. Slack provides it via `conversations.list` — or so we thought.

## The silent null

Claude caught this in the final code review: Slack's `conversations.list` does *not* return `num_members` by default. You have to pass `include_num_members=true` explicitly. The WireMock test stubs included the field in the response JSON, so the tests passed happily. Production would have returned null for every channel's member count, with no error, no warning.

Web search results confidently state the field is returned by default. The Slack docs show it in example responses. Neither is accurate. The parameter is required, the field is absent without it, and the failure is completely silent.
