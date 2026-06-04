---
layout: post
title: "Opening connectors to the LLM ecosystem"
date: 2026-06-04
type: phase-update
entry_type: note
subtype: diary
projects: [casehub-connectors]
tags: [mcp, quarkus, architecture]
---

connectors#1 was filed as a conformance task — XS, Low complexity. Add a section to DESIGN.md documenting which message types agents should use when talking to Qhorus channels. Done in an afternoon.

I spent two sessions on it instead, and shipped a five-tool MCP surface.

The pivot happened about twenty minutes in. Scanning the codebase to understand what "conformance" actually meant, I found that casehub-connectors has no Qhorus code at all — the bridge lives in qhorus, not here. The conformance rules apply to code that doesn't yet exist. Writing documentation for code that doesn't exist and that nobody has asked for felt like the wrong use of time.

The more useful question was: who would actually use connector MCP tools, and for what? The answer that made sense was the broader LLM ecosystem — not just CaseHub-managed agents, but any developer who wants to send a Slack notification, an SMS, or an email from their LLM agent without writing connector infrastructure from scratch. casehub-connectors has solid implementations — constant-time HMAC verification, at-least-once delivery semantics, proper edge-case handling — but they're locked behind CDI. MCP is the door out.

The second insight was about Qhorus. Standalone connector tools would get people using the library. Then when they wanted visibility — who sent what, when, what obligation was created — that's Qhorus. The connector tools include a `ConnectorMeshBridge` SPI that does nothing by default. When `qhorus/connector-backend` implements it and lands on the classpath, every connector delivery shows up as an `EVENT` on the observe channel. The developer doesn't change any code.

The architectural decision that mattered was where `ConnectorMeshBridge` lives. I put it in `core`, not in the new `mcp` module. The reason: `qhorus/connector-backend` already depends on `casehub-connectors-core` — if the SPI lived in `mcp`, qhorus would have to take a new dep on `quarkus-mcp-server-core` just to provide a CDI implementation. That's MCP infrastructure with no value to a module that isn't exposing any MCP tools. The price of keeping the SPI in `core` is an `@Unremovable` annotation on the no-op default — ARC sees no injection point in `core` itself, since the injection point lives in `mcp`, and would silently eliminate the bean at augmentation time.

We built five tools: `send_slack`, `send_teams`, `send_sms`, `send_whatsapp`, `send_email`. The code review process caught two things worth noting. The first: `ConnectorService.send()` is void. We had written the tools to return `"Delivered to <destination>"` on success, which is a lie — the connector logs failures internally and returns silently. Changed to `"Dispatched to"`. It matters because LLMs act on what tools tell them; a false delivery confirmation is an actionable false premise.

The second: the content sanitizer started as three chained `String.replace()` calls. Claude flagged that this misses ESC (`\x1B`) — the most dangerous control character for ANSI injection in terminal-rendered logs — along with NUL, BEL, and the rest of the ASCII control range. We replaced it with a single-pass `StringBuilder` loop using `c < 0x20 || c == 0x7F` that handles all ASCII control characters and truncation in one iteration.

WhatsApp needed template support. The API requires pre-approved templates for first-contact messages and messages outside the 24-hour engagement window. We extracted `buildPayload()` from `WhatsAppConnector.send()` as a static method and branched on `ConnectorMessage.attributes("templateName")`. `send_whatsapp` exposes both `templateName` and `templateLanguage` as optional MCP parameters. The qhorus bridge, when it lands, will post `EVENT` using the sanitized body — not raw content, since email bodies and WhatsApp messages can carry PII.

connectors#15, whenever it arrives, will be the Qhorus bridge itself. Everything is ready on this side.
