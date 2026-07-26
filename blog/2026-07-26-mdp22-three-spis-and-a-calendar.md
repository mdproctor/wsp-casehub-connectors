---
title: "Three SPIs and a Calendar"
date: 2026-07-26
entry_type: note
subtype: diary
projects: [casehub-connectors]
tags: [notification-bridge, calendar, spi, mcp, digest]
---

# Three SPIs and a Calendar

The notification bridge shipped last session but couldn't deliver anything — no resolver
existed to map users to destinations. The `DestinationResolver` SPI was in platform-api,
`NotificationBridgeStartup` wired everything together, but the actual "user-1 gets email
at user1@example.com" mapping was absent. Today I closed that gap and two adjacent issues
on the same branch.

## Config resolver as fallback

The resolver design went through an interesting iteration. My first instinct was a CDI
producer creating one resolver bean per channel type. Claude and I discovered that standard
CDI can't dynamically produce N beans from a single `@Produces` method — each producer
defines exactly one bean instance. The design review caught this before implementation
started.

The fix was simpler: `NotificationBridgeStartup` already does the wiring and is the
only consumer of resolvers. It scans `casehub.notification.destinations.<channel>.<userId>`
config properties and creates a `ConfigDestinationResolver` directly as a fallback when
no CDI-provided resolver exists. CDI resolvers always take precedence — the config
fallback activates only when nothing else handles that channel type. No hardcoded channel
list, no producer, no CDI magic.

## Digest delivery and the HTML question

`ConnectorNotificationDeliverer.deliverDigest()` was a hard-coded failure stub. The
issue asked for channel-type-aware formatting — HTML for email, short text for SMS,
rich text for WhatsApp.

I made `DigestFormatter` a CDI SPI rather than a static utility. The design review pushed
this — a static registry contradicts the CDI-native architecture and prevents downstream
applications from overriding formatters. Each formatter is an `@ApplicationScoped` bean
matched by `channelId()`. The startup indexes them and passes the matching one to each
deliverer.

The email formatter writes HTML. But `EmailConnector.send()` called `Mail.withText()`
unconditionally — the HTML would have been delivered as plain text, angle brackets and all.
A `format=html` attribute on `ConnectorMessage` signals `EmailConnector` to use
`Mail.withHtml()` instead. Backward-compatible — default remains plain text.

## CalendarPlatform — following the ChatPlatform pattern

Issue #88 is the biggest piece. The `casehub-life` OpenClaw spec needs agents to interact
with calendars. I followed the ChatPlatform module pattern: `calendar-spi` for the
interface, `calendar-ref` for in-memory testing, `calendar-google` for the real provider.

The interface is flat — no capability decomposition. ChatPlatform has nine sub-interfaces
because chat platforms genuinely vary in what they support. Calendar systems are more
uniform. If a future provider needs degradation, we break the interface then. Pre-release,
that's free.

The event timing model was the most interesting design question. My initial draft used
`boolean allDay` with `Instant` start/end. The design review caught the problem: all-day
events are date-scoped, not instant-scoped. Converting "July 27th all day" to an `Instant`
requires picking a timezone and a midnight — lossy in both directions. A sealed
`EventTiming` interface with `Timed(Instant, Instant, ZoneId)` and
`AllDay(LocalDate, LocalDate)` encodes the distinction at the type level. `AllDay.end` is
exclusive per iCalendar RFC 5545 — a one-day event on July 27 has end July 28.

The Google Calendar provider wraps `google-api-services-calendar` directly rather than
depending on jgccli. jgccli is a CLI tool with its own account management and Picocli
infrastructure — baggage we don't need. The API calls are the same; the provider just
needs OAuth2 credentials from config.

The MCP tool layer provides PATCH convenience on top of the SPI's PUT semantics. Agents
call `calendar_update_event(summary="New title")` without providing every other field.
The tool fetches the existing event, merges the provided parameters, and calls the SPI
with the complete state.

## What's next

The bridge is functional for dev/test. Production needs a real resolver — SCIM, database,
or OIDC-claims-backed. Per-tenant destination deduplication (#90) unblocks Slack and Teams
bridging. The calendar SPI is ready for the OpenClaw integration in casehub-life.
