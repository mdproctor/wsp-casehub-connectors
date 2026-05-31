---
layout: post
title: "IMAP IDLE and the attachment we forgot to ship"
date: 2026-05-31
type: phase-update
entry_type: note
subtype: diary
projects: [casehub-connectors]
tags: [imap, email, jakarta-mail, attachments]
---

The polling model was always a placeholder. Open a connection, search unseen
messages, close, wait 60 seconds, repeat. It worked, but latency of up to a
minute is the wrong answer for a platform routing messages to AI agents.

Issue #9 was about fixing that with IMAP IDLE. I brought Claude in for the
implementation. Before a line was written we argued the design: commit fully to
IDLE, or build a polling fallback? RFC 2177 is 29 years old. Gmail, Outlook
365, Fastmail, Dovecot, Exchange — all support it. A fallback would be permanent
maintenance burden for a server capability that doesn't exist anywhere we'd
actually deploy. I went IDLE-only, loud failure if a server rejects it.

The implementation landed something unexpected: Claude flagged that the spec
called for `folder.idle(false)`, but the boolean parameter in angus-mail's
`IMAPFolder.idle(boolean once)` doesn't mean what you'd assume. `once=false`
stays in IDLE indefinitely — it only returns when another thread explicitly aborts
it. `once=true` returns after the first server notification, which is what you
want in a loop. The spec was wrong and Claude caught it. The angus-mail docs
explain it clearly once you know where to look — named counterintuitively but
documented.

## Attachments

Issue #10 was baked into the same branch. `InboundMessage` was text-only by
design ("Text content only in v1" — Javadoc written eight days earlier). The
expansion had a clear shape: a new `Attachment` record in `core` with defensive
byte-array copy on both construction and access, and a `List<Attachment>
attachments` field added to `InboundMessage` with `List.copyOf()` in the compact
constructor. Records don't enforce immutability for `byte[]` or mutable
collections — you have to do it yourself.

The `ContentExtractor` refactor was the part worth talking about. The old
implementation made two recursive passes over the MIME tree — one for
`text/plain`, one for `text/html`. We replaced it with a single pass using an
`Accumulator` that collects both text content and binary parts simultaneously.
Less traversal, and future extension doesn't require a third pass.

The full-build verification caught one real bug. `Part.getInputStream()` is the
standard documented API for reading raw bytes from a MIME part. It throws
`UnsupportedDataTypeException` for binary parts constructed in-memory via
`setContent(byte[], mimeType)`. Jakarta Activation has no DataContentHandler
registered for `application/pdf`, `image/png`, or similar types in a plain
Jakarta Mail environment. Real IMAP messages work because angus-mail decodes the
transfer-encoding before the DataHandler chain sees the bytes — in-memory test
parts hit the DCH path instead. We worked out the fix: call `Part.getContent()`
first and dispatch on the returned type — `byte[]`, `InputStream`, or `String` —
before falling back to `getInputStream()` for DCH-registered types. The forage
garden has the full writeup.

126 tests, 0 failures.
