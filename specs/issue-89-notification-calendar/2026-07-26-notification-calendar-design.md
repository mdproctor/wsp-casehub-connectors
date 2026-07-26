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

A single `ConfigDestinationResolver` class (package-private, not a CDI bean) reads userId→destination mappings from MicroProfile `Config` by scanning a key prefix. A CDI producer creates one resolver instance per bridged channel type.

**Config format:**
```properties
casehub.notification.destinations.email.user-1=user1@example.com
casehub.notification.destinations.sms.user-1=+447700900000
casehub.notification.destinations.whatsapp.user-1=+447700900000
```

**Key design decisions:**
- `resolve()` ignores `tenancyId` — single-tenant starter. Production resolvers (SCIM, database, OIDC) replace via the SPI.
- Three producer methods (email, sms, whatsapp) — one per bridged channel type. All resolvers are always present; if no config entries exist, `resolve()` returns `Optional.empty()`.
- Uses `Config.getPropertyNames()` to enumerate keys under the prefix — no `@ConfigMapping` needed, handles empty config gracefully.

### Files

| File | Purpose |
|------|---------|
| `ConfigDestinationResolver` | Package-private. Reads config prefix, provides `channelId()` + `resolve()` |
| `ConfigDestinationResolverProducer` | `@ApplicationScoped`. Three `@Produces` methods with `@Named` qualifiers |
| `ConfigDestinationResolverTest` | Unit tests: resolve hit, resolve miss, empty config, multiple channels |

All in `notification-bridge/src/main/java/io/casehub/connectors/notification/`.

---

## 3. Digest Delivery (#91)

### Problem

`ConnectorNotificationDeliverer.deliverDigest()` returns `DeliveryResult(false, "digest delivery not yet supported")`. The platform default collapses to "N new notifications" with no body or links.

### Design

`deliverDigest()` follows the same resolver→destination→send pattern as `deliver()`. Formatting varies by channel type via a `DigestFormatter` functional interface with a static registry.

```java
@FunctionalInterface
interface DigestFormatter {
    ConnectorMessage format(DigestSummary summary, String destination);
}
```

**Formatting per channel:**

| Channel | Subject/Title | Body | Grouping |
|---------|--------------|------|----------|
| email | "N notifications (period)" | Plain-text list: title, category, severity, action link per notification | Respects `groupBy == CATEGORY` |
| sms | — | "N notifications. Most urgent: \<title\>" | Always flat (no room) |
| whatsapp | — | Count + category breakdown + most urgent + action link | Always flat |
| fallback | — | "You have N notifications from \<start\> to \<end\>" | Always flat |

**Key design decisions:**
- Formatters live in a separate `DigestFormatters` utility class — testable independently.
- Email body is plain text, not HTML. `ConnectorMessage.body` is a plain string; the email connector decides rendering format.
- Unknown channel types get a plain fallback, not the platform default (which is misleading).

### Files

| File | Purpose |
|------|---------|
| `DigestFormatter` | Functional interface (package-private) |
| `DigestFormatters` | Static formatter methods per channel type (package-private) |
| `ConnectorNotificationDeliverer` | Updated `deliverDigest()` implementation |
| `ConnectorNotificationDelivererTest` | New digest tests: email format, sms format, whatsapp format, fallback, groupBy, no resolver, no destination |

All in `notification-bridge`.

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
    CalendarEvent createEvent(String calendarId, CreateEventRequest request);
    void deleteEvent(String calendarId, String eventId);
}
```

`id()` follows the `spi-id-method-naming` protocol.

`listCalendars()` and `getEvent()` are beyond the issue's stated three operations but trivial to implement and genuinely needed by agents.

### Model records

```java
public record CalendarInfo(
    String id, String summary, String description, boolean primary) {}

public record CalendarEvent(
    String id, String calendarId, String summary, String description,
    String location, Instant start, Instant end, boolean allDay,
    List<String> attendees) {}

public record CreateEventRequest(
    String summary, String description, String location,
    Instant start, Instant end, boolean allDay,
    List<String> attendees) {}
```

Platform-agnostic — no Google types leak through.

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

**Authentication:** OAuth2 refresh token in config. `GoogleCalendarPlatform` is the caller of the Google API, not a shared client — credential-config-ownership protocol does not apply.

```properties
casehub.connectors.calendar.google.client-id=...
casehub.connectors.calendar.google.client-secret=...
casehub.connectors.calendar.google.refresh-token=...
```

Google Calendar API client built at startup. Single-account for now.

### RefCalendarPlatform

In-memory `ConcurrentHashMap<String, List<CalendarEvent>>`. `@ApplicationScoped`, `id() = "ref"`. Same approach as `RefChatPlatform` with `ChatBackend`.

### MCP tools

`CalendarMcpTool` in existing `mcp/` module. All `@Tool` methods annotated `@Blocking` per protocol.

| Tool | Parameters | Notes |
|------|-----------|-------|
| `calendar_list_calendars` | `platform` | Discovery |
| `calendar_list_events` | `platform, calendarId, from, to` | `calendarId` defaults to `"primary"` |
| `calendar_create_event` | `platform, calendarId, summary, description, location, start, end, allDay, attendees` | Returns created event details |
| `calendar_delete_event` | `platform, calendarId, eventId` | Confirmation message |

Routes through `CalendarPlatformService` — same pattern as `ChatPlatformMcpTool`.

---

## 5. Cross-repo impact

None. All three issues are self-contained in connectors:
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
| `credential-config-ownership` | GoogleCalendarPlatform | N/A — not a shared client |
| `paginating-client-fail-soft` | CalendarPlatform.listEvents | N/A — single API call, no pagination loop |

---

## 7. Testing strategy

| Issue | Test approach |
|-------|-------------|
| #89 | Unit: resolve hit/miss, empty config, multiple channels. Integration: full bridge wiring with config resolver |
| #91 | Unit: each formatter independently (email, sms, whatsapp, fallback). Unit: deliverDigest flow (resolver, destination, send). Edge: groupBy variants, empty notifications list (rejected by DigestSummary constructor) |
| #88 SPI | Contract test in `calendar-spi` — same pattern as `DestinationResolverContractTest` |
| #88 Ref | `RefCalendarPlatformContractTest` — runs the contract test against the reference impl |
| #88 Google | Unit with WireMock — mock Google Calendar API responses. Verify model translation. |
| #88 MCP | Unit: tool methods with mock CalendarPlatformService |

---

## 8. Not in scope

- CalendarPlatform capability decomposition (sub-interfaces, `supports()`, degradation)
- `updateEvent`, `freeBusy`, ACL operations
- Multi-account Google Calendar
- OpenClaw fallback calendar provider
- HTML email formatting for digests
- Production destination resolvers (SCIM, database, OIDC)
