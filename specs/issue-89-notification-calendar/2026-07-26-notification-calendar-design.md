# Notification Bridge Completion + CalendarPlatform SPI — Design Spec

**Issues:** #89, #91, #88
**Date:** 2026-07-26
**Status:** Approved

---

## 1. Overview

Three pieces on one branch:

1. **#89** — Config-based `DestinationResolver` in `notification-bridge`. Makes the bridge functional for dev/test.
2. **#91** — Channel-type-aware digest delivery in `ConnectorNotificationDeliverer`. Replaces the hard-coded failure.
3. **#88** — `CalendarPlatform` SPI + Google Calendar provider + MCP tools. Follows the `ChatPlatform` pattern.

---

## 2. Config-based DestinationResolver (#89)

### Problem

`NotificationBridgeStartup` wires connectors to resolvers via `@All List<DestinationResolver>`, but no implementation exists. Delivery always fails with "no destination resolver."

### Design

A single `ConfigDestinationResolver` class (package-private, not a CDI bean) reads userId→destination mappings from MicroProfile `Config` by scanning a key prefix.

`NotificationBridgeStartup` creates `ConfigDestinationResolver` instances directly as a fallback for any connector whose `channelType()` has config entries but no CDI-provided `DestinationResolver`. This eliminates the need for a CDI producer — the startup already does the wiring and is the only consumer of resolvers.

**Config format:**
```properties
casehub.notification.destinations.email.user-1=user1@example.com
casehub.notification.destinations.sms.user-1=+447700900000
casehub.notification.destinations.whatsapp.user-1=+447700900000
```

**Wiring in `NotificationBridgeStartup.registerBridgedChannels()`:**
```java
DestinationResolver resolver = resolverIndex.get(channelType);
if (resolver == null) {
    var configResolver = new ConfigDestinationResolver(channelType, config);
    if (configResolver.hasEntries()) {
        resolver = configResolver;
    }
}
```

CDI-provided resolvers (from production implementations like SCIM or database) take precedence — the config fallback only activates when no CDI bean exists for that channel type.

**Key design decisions:**
- `resolve()` ignores `tenancyId` — single-tenant starter. Production resolvers (SCIM, database, OIDC) replace via the SPI.
- No hardcoded channel list. The startup creates a config resolver for any channel type that has entries under `casehub.notification.destinations.<channelType>.*`. Adding a new connector with a `channelType()` only requires config entries.
- No CDI producer needed. `ConfigDestinationResolver` is package-private and only used by `NotificationBridgeStartup`. Standard CDI cannot dynamically produce N beans from a single `@Produces` method, and the startup is the sole consumer — direct instantiation is simpler and correct.
- If no config entries exist for a channel type and no CDI resolver exists, `resolver` stays null and `ConnectorNotificationDeliverer` handles it gracefully (returns `DeliveryResult(false, "no destination resolver for <channelType>")`).

### Files

| File | Purpose |
|------|---------|
| `ConfigDestinationResolver` | Package-private. `ConfigDestinationResolver(String channelType, Config config)`. Reads config prefix, provides `channelId()` + `resolve()` + `hasEntries()` |
| `NotificationBridgeStartup` | Updated `registerBridgedChannels()` — creates config resolvers as fallback for connectors without a CDI-provided resolver |
| `ConfigDestinationResolverTest` | Unit tests: resolve hit, resolve miss, empty config, multiple channels |
| `NotificationBridgeStartupTest` | Updated: config resolver fallback, CDI resolver precedence, no config no CDI |

All in `notification-bridge/src/main/java/io/casehub/connectors/notification/`.

---

## 3. Digest Delivery (#91)

### Problem

`ConnectorNotificationDeliverer.deliverDigest()` returns `DeliveryResult(false, "digest delivery not yet supported")`. The platform default collapses to "N new notifications" with no body or links.

### Design

`deliverDigest()` follows the same resolver→destination→send pattern as `deliver()`. Formatting varies by channel type via `DigestFormatter` — a CDI SPI discovered via `@All List<DigestFormatter>`, indexed by `channelId()` in `ConnectorNotificationDeliverer`.

```java
public interface DigestFormatter {
    String channelId();
    ConnectorMessage format(DigestSummary summary, String destination);
}
```

Each `DigestFormatter` is an `@ApplicationScoped` CDI bean. Implementations are matched to deliverers by `channelId()`. If no formatter exists for a channel type, a `DefaultDigestFormatter` provides a plain fallback. Downstream applications can override any formatter by providing a higher-priority CDI bean with the same `channelId()`.

**Formatting per channel:**

| Channel | Subject/Title | Body | Grouping |
|---------|--------------|------|----------|
| email | "N notifications (period)" | HTML summary: title, category, severity, action link per notification. Sets `attributes.put("format", "html")` on the `ConnectorMessage`. | Respects `groupBy == CATEGORY` |
| sms | — | "N notifications. Most urgent: \<title\>" | Always flat (no room) |
| whatsapp | — | Count + category breakdown + most urgent + action link | Always flat |
| fallback | — | "You have N notifications from \<start\> to \<end\>" | Always flat |

**Grouping strategy handling:**
- `FLAT` — all notifications in a single list
- `CATEGORY` — notifications grouped under category headings (email only; other channels always flat)
- `ENTITY` — treated as `FLAT` with a `WARNING` log. Entity-based grouping requires entity metadata (type descriptions, display names) that the formatter does not have. Tracked as a future enhancement.

**EmailConnector HTML support:**

`EmailConnector.send()` currently calls `Mail.withText()` unconditionally and ignores `ConnectorMessage.attributes()`. This must be modified to support HTML digest delivery:

```java
public boolean send(final ConnectorMessage message) {
    // ... existing validation ...
    String format = message.attributes() != null
            ? message.attributes().getOrDefault("format", "text") : "text";
    if ("html".equals(format)) {
        mailer.send(Mail.withHtml(message.destination(), subject, body));
    } else {
        mailer.send(Mail.withText(message.destination(), subject, body));
    }
}
```

`ConnectorMessage` Javadoc updated to document the `format` attribute: `"html"` causes `EmailConnector` to use `Mail.withHtml()` instead of `Mail.withText()`. Default is plain text — backward-compatible.

**Key design decisions:**
- DigestFormatter is a CDI SPI following the `Connector` pattern — `@ApplicationScoped` beans discovered via `@All List<DigestFormatter>`, indexed by `channelId()`.
- Email body is HTML, matching issue #91's requirement ("Email: HTML summary with links"). `ConnectorMessage.body` is a String — HTML is valid content. The `format=html` attribute signals `EmailConnector` to use Quarkus Mailer's `Mail.withHtml()`.
- Unknown channel types get a plain fallback via `DefaultDigestFormatter`, not the platform default (which is misleading).

### Files

| File | Purpose |
|------|---------|
| `DigestFormatter` | CDI SPI interface — `channelId()` + `format()` |
| `EmailDigestFormatter` | `@ApplicationScoped`. HTML formatting with category grouping. Sets `format=html` attribute |
| `SmsDigestFormatter` | `@ApplicationScoped`. Count + most urgent |
| `WhatsAppDigestFormatter` | `@ApplicationScoped`. Count + category breakdown |
| `DefaultDigestFormatter` | `@ApplicationScoped @DefaultBean`. Plain fallback for unknown channel types |
| `ConnectorNotificationDeliverer` | Updated `deliverDigest()` implementation — looks up formatter by channel type |
| `EmailConnector` | Updated `send()` — checks `attributes.get("format")`, uses `Mail.withHtml()` for `"html"` |
| `ConnectorNotificationDelivererTest` | New digest tests: email format (HTML), sms format, whatsapp format, fallback, groupBy (CATEGORY, ENTITY warning), no resolver, no destination |
| `EmailConnectorTest` | New test: HTML rendering via `format=html` attribute, backward-compatible plain text default |

`notification-bridge` files + `EmailConnector`/`EmailConnectorTest` in `email` module.

---

## 4. CalendarPlatform SPI (#88)

### Problem

Agents need calendar access. The `ChatPlatform` SPI pattern is established. `CalendarPlatform` follows the same shape.

### Interface — flat, not decomposed

ChatPlatform has 9 capability sub-interfaces because chat platforms vary in what they support. Calendar is more uniform. A flat interface is the right design. If a future provider needs degradation (read-only iCal feeds), we break the interface then — pre-release, that's free.

```java
public interface CalendarPlatform {
    String id();
    List<CalendarInfo> listCalendars();
    List<CalendarEvent> listEvents(String calendarId, Instant from, Instant to);
    CalendarEvent getEvent(String calendarId, String eventId);
    CalendarEvent createEvent(String calendarId, EventDetails details);
    CalendarEvent updateEvent(String calendarId, String eventId, EventDetails details);
    void deleteEvent(String calendarId, String eventId);
}
```

`id()` follows the `spi-id-method-naming` protocol.

`listCalendars()` and `getEvent()` are beyond the issue's stated three operations but trivial to implement and genuinely needed by agents.

`updateEvent()` included in the initial SPI because agents need to reschedule meetings, add attendees, and change event details. With a flat interface, adding it later would break the SPI — and the need is immediate (casehub-life#60 OpenClaw skill integration). The cost is one method signature and one implementation per provider.

### Model records

```java
public sealed interface EventTiming {
    record Timed(Instant start, Instant end, ZoneId timeZone) implements EventTiming {}
    record AllDay(LocalDate start, LocalDate end) implements EventTiming {}
}

public record CalendarInfo(
    String id, String summary, String description, boolean primary) {}

public record CalendarEvent(
    String id, String calendarId, String summary, String description,
    String location, EventTiming timing,
    List<String> attendees, String recurringEventId) {}

public record EventDetails(
    String summary, String description, String location,
    EventTiming timing,
    List<String> attendees) {}
```

**Event timing model:** All-day events and timed events have incompatible temporal semantics. Google Calendar API represents them with different types (`date` vs `dateTime`). A boolean `allDay` flag with `Instant` forces a lossy round-trip through timezone-dependent midnight conversion. The sealed `EventTiming` interface encodes the distinction at the type level:
- `Timed(Instant start, Instant end, ZoneId timeZone)` — timed events carry timezone for meaningful display ("3:00 PM BST")
- `AllDay(LocalDate start, LocalDate end)` — all-day events are date-scoped, no timezone ambiguity. `end` is **exclusive** following iCalendar (RFC 5545) convention — a one-day event on June 15 has `start = 2025-06-15` and `end = 2025-06-16`. Google Calendar API uses the same convention.

Platform-agnostic — no Google types leak through.

**`EventDetails` — shared type for create and update:** Named `EventDetails` (not `CreateEventRequest`) because the same fields apply to both `createEvent()` and `updateEvent()`. Both operations use full-replacement (PUT) semantics — all fields in `EventDetails` are applied to the event. For create, all fields are fresh. For update, the caller provides the complete desired state.

**Update semantics — full replacement (PUT):** `updateEvent()` replaces the entire event with the state described in `EventDetails`. Null fields clear the corresponding value on the event. This matches Google Calendar API's `events.update` (PUT) semantics. The MCP tool layer provides PATCH convenience — see MCP tools section below.

**Recurrence:** The SPI returns expanded recurring event instances as individual `CalendarEvent` records. `recurringEventId` (nullable) indicates the event is an instance of a recurring series. Recurrence rule creation (`RRULE`) and series-level operations are not in scope for the initial SPI — tracked as future enhancement (GitHub issue to be filed).

**Recurrence behavior:**
- `listEvents()` returns expanded instances — each occurrence as a separate event with its own `id` and a non-null `recurringEventId` pointing to the series
- `getEvent()` returns a single instance; `recurringEventId` indicates series membership
- `deleteEvent()` on a recurring instance cancels that instance only, not the entire series
- `createEvent()` creates non-recurring events only
- `updateEvent()` on a recurring instance modifies that instance only

### Module structure

| Module | Purpose | Dependencies |
|--------|---------|-------------|
| `calendar-spi/` | CalendarPlatform interface, model records, CalendarPlatformService | Self-contained (no `core` dep) |
| `calendar-ref/` | RefCalendarPlatform — in-memory reference | `calendar-spi` |
| `calendar-google/` | GoogleCalendarPlatform — wraps Google Calendar API | `calendar-spi`, `google-api-services-calendar`, `google-api-client` |
| `mcp/` (existing) | CalendarMcpTool | `calendar-spi` (new dep) |

### CalendarPlatformService

Same routing pattern as `ChatPlatformService`: `@ApplicationScoped`, injects `@All List<CalendarPlatform>`, indexes by `id()`, throws on duplicates and unknown IDs.

### Google Calendar provider

Uses `google-api-services-calendar` and `google-api-client` directly — same libraries jgccli uses, without its CLI/account-management baggage. Translates between platform model records and Google API types.

**Authentication:** OAuth2 refresh token in config.

```properties
casehub.connectors.calendar.google.client-id=...
casehub.connectors.calendar.google.client-secret=...
casehub.connectors.calendar.google.refresh-token=...
```

Google Calendar API client built at startup. Single-account for now.

**Credential fail-soft:** If any credential property is blank, `GoogleCalendarPlatform` logs WARNING at startup and does not build the API client. `CalendarPlatformService` simply does not include a `"google"` platform. This follows the established L1 pattern (`TwilioSmsConnector`, `WhatsAppConnector`): blank credentials → connector included but inactive, safe in any deployment profile.

**Pagination:** `GoogleCalendarPlatform.listEvents()` handles pagination internally following the `parsePage()` + cursor loop pattern from `SlackBotClient.listChannels()` per protocol PP-20260610-83747b. Google Calendar API `events.list` paginates via `nextPageToken` (default 250 events/page, max 2500). The SPI contract is that `listEvents()` returns all matching events — implementations handle pagination transparently. On mid-loop failure, partial results are returned with a WARNING log.

### RefCalendarPlatform

In-memory `ConcurrentHashMap<String, List<CalendarEvent>>`. `@ApplicationScoped`, `id() = "ref"`. Same approach as `RefChatPlatform` with `ChatBackend`.

### MCP tools

`CalendarMcpTool` in existing `mcp/` module. All `@Tool` methods annotated `@Blocking` per protocol.

| Tool | Parameters | Notes |
|------|-----------|-------|
| `calendar_list_calendars` | `platform` | Discovery |
| `calendar_list_events` | `platform, calendarId, from, to` | `calendarId` defaults to `"primary"` |
| `calendar_get_event` | `platform, calendarId, eventId` | Returns full event details including timing and recurrence info |
| `calendar_create_event` | `platform, calendarId, summary, description, location, start, end, timeZone, startDate, endDate, attendees` | See timing validation rules below. Returns created event details |
| `calendar_update_event` | `platform, calendarId, eventId, summary, description, location, start, end, timeZone, startDate, endDate, attendees` | PATCH convenience — fetches existing event, merges provided parameters, calls SPI `updateEvent()` with full state. See timing and update rules below |
| `calendar_delete_event` | `platform, calendarId, eventId` | Confirmation message |

Routes through `CalendarPlatformService` — same pattern as `ChatPlatformMcpTool`.

**MCP tool timing parameter validation (`calendar_create_event`, `calendar_update_event`):**

The tool accepts flat parameters that map to `EventTiming`. Validation follows the fail-soft MCP tool pattern — invalid combinations return `"Failed: ..."`, never throw.

| Condition | Result |
|-----------|--------|
| Both `start` and `startDate` provided | `"Failed: provide either start/end/timeZone (timed) or startDate/endDate (all-day), not both"` |
| `start` provided without `timeZone` | `"Failed: timeZone is required for timed events (e.g. 'Europe/London', 'America/New_York')"` |
| `start` provided without `end` | `"Failed: end is required when start is provided"` |
| `startDate` provided without `endDate` | `"Failed: endDate is required when startDate is provided"` |
| `end` provided without `start` | `"Failed: start is required when end is provided"` |
| Neither `start` nor `startDate` for create | `"Failed: timing is required — provide start/end/timeZone or startDate/endDate"` |
| Neither `start` nor `startDate` for update | Valid — timing unchanged, only other fields updated |
| All parameters null for update | `"Failed: at least one field must be provided for update"` |

**MCP tool update semantics (`calendar_update_event`):**

The MCP tool provides PATCH convenience on top of the SPI's PUT semantics. The tool:
1. Calls `getEvent()` to fetch the current state
2. Merges provided parameters over the existing values — only non-null parameters override
3. Calls `updateEvent()` with the merged `EventDetails`

This means agents can call `calendar_update_event(eventId=..., summary="New title")` without providing every other field. The tool handles the get-merge-put cycle internally.

---

## 5. Cross-repo impact

`EmailConnector` in the `email` module is modified (§3) to support the `format=html` attribute. This is a backward-compatible change — the default remains plain text.

All other changes are self-contained in connectors:
- **#89:** `DestinationResolver` SPI already exists in `casehub-platform-api`. Config resolver is a new implementation.
- **#91:** `DigestSummary`, `NotificationDeliverer` already exist in `casehub-platform-api` with the right shape.
- **#88:** `CalendarPlatform` SPI is connector-local (same as `ChatPlatform`). No platform-api changes.

---

## 6. Protocols checked

| Protocol | Applies to | Status |
|----------|-----------|--------|
| `spi-id-method-naming` | CalendarPlatform | `id()` — compliant |
| `shared-http-client` | GoogleCalendarPlatform | N/A — uses Google's own HTTP transport, not `HttpHelper.CLIENT` |
| `mcp-tool-blocking-annotation` | CalendarMcpTool | All `@Tool` methods will be `@Blocking` |
| `credential-config-ownership` | GoogleCalendarPlatform | N/A — not a shared HTTP client used by multiple consumers with different credential needs (protocol governs shared clients like `SlackBotClient`). Credential fail-soft behaviour documented separately in §4 following the L1 pattern. |
| `paginating-client-fail-soft` | GoogleCalendarPlatform.listEvents | Compliant — `parsePage()` + cursor loop with fail-soft partial return, following `SlackBotClient.listChannels()` pattern. `MAX_PAGES` cap with distinct WARNING. |

---

## 7. Testing strategy

| Issue | Test approach |
|-------|-------------|
| #89 | Unit: resolve hit/miss, empty config, multiple channels. Integration: full bridge wiring with config resolver fallback, CDI resolver precedence |
| #91 formatters | Unit: each formatter independently (email HTML, sms, whatsapp, fallback). Edge: groupBy variants (FLAT, CATEGORY, ENTITY warning) |
| #91 delivery | Unit: deliverDigest flow (resolver, destination, send). Edge: no resolver, no destination, empty notifications (rejected by constructor) |
| #91 EmailConnector | Unit: `format=html` attribute → `Mail.withHtml()`. Unit: default (no attribute) → `Mail.withText()` (backward-compatible) |
| #88 SPI | Contract test in `calendar-spi` — same pattern as `DestinationResolverContractTest` |
| #88 Ref | `RefCalendarPlatformContractTest` — runs the contract test against the reference impl |
| #88 Google | Unit with WireMock — mock Google Calendar API responses. Verify model translation, pagination, all-day vs timed event mapping |
| #88 MCP | Unit: tool methods with mock CalendarPlatformService. Edge: timing parameter validation (both provided, missing timeZone, missing end), update merge behavior |

---

## 8. Not in scope

- CalendarPlatform capability decomposition (sub-interfaces, `supports()`, degradation)
- `freeBusy`, ACL operations
- Recurrence rule creation (`RRULE`) and series-level delete
- Multi-account Google Calendar
- OpenClaw fallback calendar provider
- Production destination resolvers (SCIM, database, OIDC)
- `DigestGroupBy.ENTITY` formatting (treated as FLAT with warning)

---

## 9. Deliverables beyond code

- **ARC42STORIES.MD update:** New layer entry (L12 — Calendar Platform) covering `calendar-spi`, `calendar-ref`, `calendar-google` modules. Module table updates in §5. New chapter entry in §9 for the Calendar Platform journey. MCP tools table updated in L4/L10 entries.
