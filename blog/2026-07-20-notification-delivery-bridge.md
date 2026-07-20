---
title: "Bridging Notifications to Connectors"
date: 2026-07-20
type: diary
tags: [connectors, notifications, platform, SPI, design]
---

The platform notification system and the connector library evolved independently. Notifications had `NotificationDeliverer` and a registry — but only in-app inbox was wired up. Connectors had `Connector.send()` with five built-in implementations (email, SMS, Slack, Teams, WhatsApp) — but no connection to the notification pipeline. Two systems that should have been one seam.

The bridge itself is straightforward: iterate all `Connector` CDI beans at startup, wrap each in a `NotificationDeliverer`, register it. The interesting design questions were elsewhere.

## Channel Types Are Not Connector IDs

The first instinct was to use `Connector.id()` as the notification channel identifier. That's wrong. `id()` is implementation-specific — `"twilio-sms"` identifies the Twilio implementation, not the SMS channel. If you swap Twilio for Vonage, user preferences shouldn't break.

I looked at how Knock, MagicBell, and Novu model this. They converge on a three-concept model: **channel** (the delivery medium — email, SMS, push), **provider** (the swappable implementation — SendGrid, Twilio, FCM), and **subscriber profile** (user contact attributes stored separately from notification preferences). The channel/provider split is universal. Contact attributes as a separate concern from notification preferences is also universal — Novu is the most explicit about it, with structured fields on the subscriber entity.

There's a subtlety the generic model misses: not all channels are medium types. SMS and email are delivery media — you can swap the provider transparently. But Slack and Teams are platforms with their own capabilities, audiences, and user preferences. A user who enables "Slack notifications" has a different intent from one who enables "SMS notifications" — the first is choosing a platform, the second is choosing a medium. The channel taxonomy needs to reflect this: medium-type channels (SMS, email) where providers are swappable, and platform channels (Slack, Teams, WhatsApp) where each platform is its own channel.

`Connector` now has `channelType()` — defaults to `id()`, override to remap (`TwilioSmsConnector` returns `"sms"`), or return `null` to opt out.

## The Destination Gap

`NotificationInput` carries a `userId`. `ConnectorMessage` needs a `destination` — an email address, phone number, or webhook URL. The platform has no user directory with contact attributes. No SCIM `/Users` endpoint, no user profile store. The SCIM module only has Groups.

`DestinationResolver` fills this seam — a per-channel CDI SPI in `casehub-platform-api` that resolves `(userId, tenancyId) → Optional<String>`. It lives in platform-api (not the bridge module) because destination resolution is a platform-level concern. Any future `NotificationDeliverer`, connector-backed or not, needs the same capability.

The resolver is the integration point for whatever user store eventually exists — SCIM, OIDC claims, a database, configuration. The bridge doesn't care which.

## Per-Tenant Destinations Are Deferred

Slack and Teams use webhook URLs that target channels, not users. When a subscription matches 20 users and the dispatcher calls `deliver()` 20 times, a per-user channel (email) sends 20 different emails — correct. A per-tenant channel (Slack) posts the same message to the same webhook 20 times — wrong.

The deduplication can't happen in the bridge — `deliver()` is called per-user with no visibility into other users' destinations. It has to happen in the dispatcher, which sees all recipients. Until that's built, Slack and Teams opt out of notification bridging by returning `null` from `channelType()`. They remain fully functional as connectors for MCP tools and direct `ConnectorService` use.

## send() Returns Boolean Now

A side effect of the bridge work: `Connector.send()` changed from `void` to `boolean`. The void contract made `DeliveryResult` meaningless — the bridge couldn't know whether the connector succeeded. Every implementation already tracked success internally (HTTP status checks, try/catch), they just swallowed the result. Pre-release platform, so the breaking change costs nothing. Callers that discard the return value compile unchanged.
