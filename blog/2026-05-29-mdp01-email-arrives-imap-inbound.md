---
layout: post
title: "Email arrives — IMAP inbound and the spec that took four rounds"
date: 2026-05-29
type: phase-update
entry_type: note
subtype: diary
projects: [casehub-connectors]
tags: [imap, email-inbound, greenmail, jakarta-mail, webhooks]
---

Four review rounds on the email inbound spec before a line of implementation
code. Over twenty findings. Three of them would have caused real problems in
production. I'm glad we ran them.

I brought Claude into this branch for the full stack: design review, TDD,
implementation, and the leftover v1 polish from the webhook connector work.
The split was two GitHub issues — email inbound (#7) and three deferred code
review findings (#8) from the webhook inbound SPI that shipped last session.

## The spec that took four rounds

The basic shape of the email inbound connector was clear early. A separate
`email-inbound` module — keeping angus-mail away from the quarkus-mailer
SMTP module, since they share nothing. An `EmailInboundAccountProvider` SPI
for account configuration. `ScheduledExecutorService` per account rather than
`@Scheduled`, which would have added a Quarkus extension dependency and made
the `start()`/`stop()` lifecycle methods misleading no-ops. Open/close-per-poll,
no reconnect logic, at-least-once SEEN delivery with the flag written in a
`finally` block. I knew all of that going in.

Where I got it wrong was the configuration approach. I proposed using
`casehub-platform-api` `Preferences`/`PreferenceKey` for IMAP credentials —
the same mechanism I use for runtime multi-tenant settings. Claude pushed back
in the first spec review, and the argument was good: IMAP host, port, and
password are deploy-time constants. They don't change per-tenant at runtime.
Using `Preferences` for them adds a platform dependency, requires a CDI
`PreferenceProvider` bean at runtime, and misrepresents the configurability
of the thing. `@ConfigProperty`, the same pattern every other connector uses,
was the right answer.

That one fix cascaded. Once `@ConfigProperty` replaced `Preferences`, the
test strategy simplified — standard Quarkus `@TestProfile` MP Config overrides
just work, no `MockPreferenceProvider` setup needed. Three of the spec review
findings downstream of that choice resolved themselves.

The other findings were more structural. `connectorId` had drifted from its
contract — I'd proposed putting the account id there in multi-account
deployments, which would have broken observer routing. The fix was simple:
`connectorId` is always the connector type string (e.g. `"email-inbound"`),
per-account identity goes in `metadata["account-id"]`. `externalChannelRef`
was initially mapped to the IMAP folder name — "INBOX" — which carries no
routing information whatsoever. It should be the recipient email address:
where the message arrived, not where it's stored. `receivedAt` was
`Instant.now()` at poll time, which could be up to 60 seconds after actual
delivery. The fallback chain (`getReceivedDate()` → `getSentDate()` →
`Instant.now()`) is straightforward but not obvious to reach for.

By rev 4, we had a clean spec. The implementation found nothing surprising.

## EmailInboundAccountProvider and ContentExtractor

The `EmailInboundAccountProvider` is a `@FunctionalInterface` returning
`List<EmailInboundAccount>`. The `@DefaultBean` reads from `@ConfigProperty`.
Multi-account deployments override with a custom CDI bean. No complexity for
the common case.

The `ContentExtractor` handles recursive MIME structures. The naive approach
— check the top-level content type — fails silently for any email with an
attachment:

```
multipart/mixed
  └── multipart/alternative
        ├── text/plain  ← want this
        └── text/html
  └── application/pdf   ← ignore
```

The top-level type is `multipart/mixed`, which isn't `text/plain` or
`text/html`, so the naive extractor returns empty string and silently discards
the body. We wrote the recursive descent — prefer `text/plain` depth-first,
fall back to `text/html`, fall back to `""` — and five test cases before
touching the connector code.

## Greenmail and its API surprises

Testing IMAP-based code against an embedded server has opinions about what
you're allowed to do.

`GreenMailExtension` (the JUnit 5 wrapper) doesn't expose `deliver(MimeMessage)`.
The underlying `GreenMail` class does, but `getGreenMail()` is declared
`protected` — not callable from test code. Discoverable only via `javap` on
the jar; nothing in the Greenmail documentation mentions this. We ended up
using `Transport.send()` via SMTP for most tests, and routing through
`greenMail.getManagers().getImapHostManager().getInbox(user).appendMessage()`
for cases where precise header control mattered.

Which matters because of the second surprise: Greenmail's SMTP pipeline
synthesises a `Message-ID` header for any message that doesn't have one.
Tests asserting absent `message-id` in the delivered message's metadata
can't go through SMTP — the header is always there by the time the message
hits the IMAP folder.

The third: `ServerSetupTest.SMTP_IMAP` uses fixed ports (SMTP=3025,
IMAP=3143). When both a `@QuarkusTestResource` and a `@RegisterExtension`
unit test extension try to bind those ports in the same Maven Surefire JVM,
the second one throws `IllegalStateException: Could not start mail server`.
The fix is `new ServerSetup(0, "localhost", "imap")` — port 0 for OS-assigned
random ports — in the unit test extension. The `@QuarkusTestResource` has to
stay on fixed ports because it needs to inject them into MP Config before
Quarkus starts.

There was also a `MimeMessage.isMimeType()` issue — covered in garden entry
GE-20260529-a488bf — where a freshly-constructed `MimeMessage` returns
`"text/plain"` from `getContentType()` regardless of what you put in it,
until you call `saveChanges()`. Tests for `ContentExtractor` that construct
multipart messages were failing with what looked like algorithm bugs. They
weren't.

## v1 polish: three improvements

Issue #8 was three deferred findings from the webhook inbound work.

`WebhookRouter`'s rejection log was a flat string. We rewrote it as key=value
fields and added the `X-Forwarded-For` source IP. Then Claude flagged during
code review that `X-Forwarded-For` is caller-controlled — an attacker can
inject newlines to create fake log lines, or forge a legitimate-looking IP.
The fix: validate with a regex and take only the first comma-separated segment
before logging. That became PP-20260529-ab32a9 in the platform protocols.

`SlackInboundConnector.doHandle()` was parsing the request body twice for
`url_verification` events — once to check the type, once to extract the
challenge value. We replaced both methods with a single `extractChallenge()`
returning `Optional<String>`. One parse, returns the challenge if the type
matches, empty otherwise.

The metadata finding was the most worthwhile. All four webhook connectors were
constructing `InboundMessage` with `Map.of()` for metadata — a field designed
for connector-specific message identifiers. Twilio's `MessageSid` was being
parsed and discarded. Slack's `team_id` was in the event payload, unused.
WhatsApp's message `id` was available. We populated `workspace-id` for Slack,
`message-sid` for Twilio, `message-id` for WhatsApp. Teams had nothing
relevant. Now the email inbound connector isn't the only one using the
metadata field.

## 108 tests, clean build

The full project — core, webhook, email, email-inbound — builds to 108 tests
passing. Email-inbound accounts for 24: five for `ContentExtractor`, four for
`DefaultEmailInboundAccountProvider`, fourteen for the connector itself (twelve
against Greenmail, two direct `buildMetadata()` tests that bypass IMAP to cover
the absent-header branches Greenmail can't test), and one `@QuarkusTest`
integration test that sends via SMTP, waits up to five seconds for the poll
cycle, and asserts the CDI event fired with correct fields.

Six garden entries. Three protocols. The connectors repo has its first inbound
email capability — IMAP polling, account SPI, recursive MIME extraction,
at-least-once delivery.
