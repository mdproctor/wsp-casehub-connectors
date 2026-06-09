---
layout: post
title: "Tools for the Bot"
date: 2026-06-09
type: phase-update
entry_type: note
subtype: diary
projects: [casehub-connectors]
tags: [mcp, slack, spi, quarkus]
---

The previous entry shipped `SlackBotClient` — a clean HTTP client for the Slack Web API.
This one wires it into the MCP surface so an LLM agent can actually use it.

The obvious design was two Slack-specific tools: `send_slack_bot` and `list_slack_channels`.
I wanted a more general shape. Channel discovery is not a Slack concern — any connector that
has a notion of named delivery targets needs it. Discord, Telegram, a demo chat server —
all would end up with their own tool if I designed Slack-first.

So `ConnectorDiscovery` landed in `core` instead: a simple SPI with `id()` and
`discover() → List<DiscoveredTarget>`. `SlackBotDiscovery` implements it; `list_channels`
aggregates all registered implementations. The next connector that needs discovery just
adds a CDI bean.

`send_slack_bot` has one design constraint that shapes everything: it bypasses `ConnectorService`.
Every other MCP tool routes through the service, which calls `Connector.send()` — void, no return.
Bot posting returns a message timestamp from Slack that an agent can save and use to reply
in-thread. Routing through a void interface throws that away. So the tool injects `SlackBotClient`
directly and returns `"Posted to C123ABC (ts=1638535627.000200)"`. The `ts` is the whole point.

There was a naming correction mid-way. The `ConnectorDiscovery` interface was drafted with
`connectorId()`, which felt natural in isolation. Code review caught it: every other SPI in this
repo uses `id()` — `Connector.id()`, `InboundConnector.id()`. The more specific name breaks
dispatch code that calls `id()` generically across SPI types. We renamed it before anything
consumed it, which is the best time to fix a naming mistake.

One fix that wasn't obvious from the commits: all five existing MCP tools were missing `@Blocking`.
Quarkus MCP server runs tool methods on the Vert.x event loop by default. Blocking HTTP on the
event loop doesn't cause visible errors in tests — it only shows up under concurrent production
load. The fix is one annotation per tool. The risk is that future tools will make the same
omission without knowing why it matters; it's now a protocol in `docs/protocols/connectors/`.

The other thing worth carrying forward: fast-fail guards in WireMock tests need more than a
return value assertion. If a blank-token guard is removed and the HTTP call fires with invalid
input, WireMock's unmatched-request error path returns the same failure string the guard would
have returned. The test passes silently. Adding `wireMock.verify(0, anyRequestedFor(anyUrl()))`
after the result check is the only assertion that actually proves the guard fired.
