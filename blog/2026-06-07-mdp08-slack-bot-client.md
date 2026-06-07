---
layout: post
title: "The Slack Bot client that almost had two connection pools"
date: 2026-06-07
type: phase-update
entry_type: note
subtype: diary
projects: [casehub-connectors]
tags: [slack, http, java]
---

The connectors side of the Slack ChannelBackend lands today — not the backend itself (that lives in qhorus), but the three pieces it depends on.

First: `InboundConnectorIds.SLACK_INBOUND`. A small constant in `connectors-core` that lets downstream modules reference the connector ID without coupling to `SlackInboundConnector` directly. The pattern already existed for Twilio and WhatsApp. Slack was the odd one out.

Second: `SlackInboundConnector` now emits `slack-ts` and `slack-thread-ts` in every delivered message's metadata, pulled from the Slack event JSON. This was the non-obvious one. The existing connector was correct — it validated signatures, filtered bots, parsed users and channels. But it silently discarded the Slack message timestamp and thread timestamp that the `SlackChannelBackend` needs to track conversations. Without those fields, the backend's thread support ships as dead code regardless of how carefully the Qhorus side is built. I missed this in the initial spec and it surfaced in the first review pass.

Third: a new Maven module, `casehub-connectors-slack-bot`, containing `SlackBotClient` — a pure `java.net.http` client for `chat.postMessage`. No Slack SDK. The decision not to use the official Bolt SDK was straightforward: OkHttp and Kotlin stdlib as transitive dependencies, known Quarkus BOM conflicts, and no path to swap the HTTP client. For an API surface this small — one POST endpoint, one response record — rolling our own was the right call.

The module split was more interesting than it looks. I could have added `SlackBotClient` to `connectors-core`, where `SlackConnector` already lives. I didn't, because `core` is a dependency of everything downstream. As soon as Block Kit formatting or reactions land in the bot client, every consumer that has never heard of the Slack Bot API picks up the baggage. `email-inbound` made the same call — split it out when the concern is heavy or direction-specific. `slack-bot` does the same.

The code review caught one thing I'd missed: `SlackBotClient` was creating `HttpClient.newHttpClient()` instead of using `HttpHelper.CLIENT`, the shared singleton already carrying the 5-second connect timeout. Two independent HTTP clients in the same process, one of them unconfigured. Claude caught it in review; we fixed it and formalised the rule as a protocol: everything in `casehub-connectors-*` that makes outbound HTTP calls goes through `HttpHelper.CLIENT`.

The Qhorus side — `SlackChannelBackend` with thread tracking, normaliser, and the `SlackBotBinding` table — is its own issue. This commit gives it what it needs.
