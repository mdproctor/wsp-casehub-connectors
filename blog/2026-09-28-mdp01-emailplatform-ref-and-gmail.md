---
layout: post
title: "EmailPlatform gets real — ref and Gmail implementations"
date: 2026-09-28
entry_type: note
subtype: diary
projects: [casehubio/connectors]
tags: [email, spi, gmail, connectors]
---

# EmailPlatform gets real — ref and Gmail implementations

The EmailPlatform SPI shipped a few weeks ago alongside BankPlatform — both as
flat interfaces with `@SimulationEligible`, both with NoOp fallbacks and service
registries. But an SPI without an implementation is a promise, not infrastructure.
Today it got two.

## email-ref — the pattern holds

The ref module follows the same shape as calendar-ref and bank-ref: a `Backend`
interface that mirrors the SPI minus `id()`, an `InMemoryEmailBackend` with
pre-loaded data, a `RefEmailPlatform` wrapper that delegates every call, and a
CDI producer. The data is realistic enough to exercise the contract — three
mailboxes (Inbox, Sent, Archive), nine messages with a mix of read/unread states,
two with attachments, and cursor-based pagination that actually pages.

The interesting constraint is the error contract. The original design spec
called for `NoSuchElementException` on not-found lookups, distinct from the
NoOp's `UnsupportedOperationException`. Bank-ref already followed this, but
calendar-ref used `IllegalArgumentException`. The spec is right — a missing
entity and a missing provider are semantically different things, and callers
need to distinguish them. Email-ref follows the spec.

## email-google — Gmail's N+1 reality

Gmail's API has a structural quirk that shapes the entire implementation.
`messages.list()` returns only message IDs and thread IDs — no headers, no
subjects, no senders. To build an `EmailSummary` with the fields consumers
actually need, each message in the page requires a separate
`messages.get()` call with `format=metadata`. That's N+1 HTTP requests per
page of results.

This is how every Gmail client works — the API was designed for batch
requests, and the standard mitigation is either batching or accepting the
round trips. I went with individual requests and per-message error handling:
if one metadata fetch fails mid-page, we log a warning and skip it rather
than failing the entire list operation. The paginating-client-fail-soft
protocol already mandates this pattern for HTTP clients that enumerate
paginated resources.

The label-to-mailbox mapping filters out Gmail's category labels
(CATEGORY_SOCIAL, CATEGORY_UPDATES) and system labels that don't map to
traditional mailbox concepts (STARRED, IMPORTANT, UNREAD). System labels
that do map get friendly names — INBOX becomes "Inbox", SENT becomes
"Sent", DRAFT becomes "Drafts". User-created labels pass through as-is.

MIME body parsing handles three shapes: single-part messages (text/plain or
text/html directly on the payload), multipart/alternative (text/plain +
text/html as sibling parts), and multipart/mixed (body parts + attachment
parts with `attachmentId`). Deeply nested MIME trees — multipart/mixed
wrapping multipart/alternative wrapping multipart/related — aren't handled
yet. That's a real-world edge case worth a follow-up, but the common
shapes cover the vast majority of email.

## The pattern tax

Both modules together are about 1,400 lines across 8 source files, 2
pom.xml files, and 33 tests. The ref module is the simpler of the two but
still carries its own interface, backend, wrapper, and producer — four
classes for what is essentially a map lookup with pre-loaded data. That's
the cost of the Backend abstraction, and it's the same cost calendar-ref
and bank-ref pay. Whether it's worth it depends on whether anyone ever
needs a non-in-memory backend for testing — a file-backed ref, say, or
one that loads data from a scenario corpus. The simulation framework may
make that unnecessary. For now, the pattern holds.
