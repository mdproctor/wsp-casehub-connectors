---
layout: post
title: "Fixing the Test That Sometimes Passed"
date: 2026-06-02
type: phase-update
entry_type: note
subtype: diary
projects: [casehub-connectors]
tags: [testing, imap, email-inbound, greenmail, awaitility]
---

The previous entry noted seventeen email-inbound tests passing clean. What I didn't say was that two of them were passing *most of the time*, which is different.

connectors#12 was the flakiness report: `messageWithNoAttachments_attachmentsEmptyAndCountIsZero` and `multipleUnseenMessages_allDelivered` failing intermittently on cold Maven builds. Error: `message not delivered within 3s — IDLE did not fire`. Pass on retry. Classic.

## Two Paths, One Race

The IMAP IDLE connector loop does this: connect → open folder → `processUnseen()` → `folder.idle(true)`. When a test calls `connector.start()` then immediately delivers a message via SMTP, two races are possible.

Path A: SMTP completes before the virtual thread finishes its IMAP handshake. When `processUnseen()` runs, the message is already there. Reliable.

Path B: the virtual thread enters `idle(true)` before SMTP completes. The connector now depends on GreenMail firing an EXISTS notification. On a warm JVM this is fast. On a cold Maven build, GreenMail's notification path — SMTP server thread → in-memory store → `Object.notifyAll()` on the IMAP folder — can take 3-4 seconds. The test was polling for exactly 3.

The fix wasn't "make the timeout bigger." That's the obvious answer and the wrong one.

## Most of These Tests Weren't Testing IDLE

I brought Claude in for the implementation. Before touching any code, I laid out which tests were actually testing the IDLE notification path and which ones happened to use SMTP incidentally.

Five tests — `htmlOnlyMessage_rawHtmlInContent`, `messageWithoutSubject_subjectKeyAbsent`, `messageWithPdfAttachment_attachmentDelivered`, `messageWithNoAttachments_attachmentsEmptyAndCountIsZero`, `messageWithMultipleAttachments_allCollected` — test content parsing, metadata, and attachment handling. None test that the IDLE notification fires. They were using SMTP delivery because that's how the test was written, not because it mattered.

Switching those five to `deliverDirect()` — GreenMail's `MailFolder.appendMessage()` — puts them on Path A. The message is in the inbox before the connector starts. `processUnseen()` finds it immediately. No IDLE involved. No race.

The remaining SMTP-after-start tests stay on SMTP because they're actually testing the notification path. `doubleStart_isNoOp` needs a live connector in IDLE. `multipleUnseenMessages_allDelivered` specifically tests two messages delivered after IDLE is established. You can't cover that with pre-delivery.

For those, we replaced `captured.poll(3, TimeUnit.SECONDS)` with Awaitility — `await().atMost(5, TimeUnit.SECONDS).untilAsserted(() -> assertThat(captured).isNotEmpty())`. Five seconds rather than three, polling continuously. If GreenMail fires at 3.2 seconds, it's caught within 100ms. The old code had already given up.

One finding: the plan specified `.failMessage("...")` on the Awaitility condition. That method doesn't exist in 4.3.0. Workaround is embedding the message in `.as(...)` inside `untilAsserted`.

## hasSize(2) and Eight Timeout Bumps

`multipleUnseenMessages_allDelivered` was originally fixed with `hasSizeGreaterThanOrEqualTo(2)` as the Awaitility condition. Claude flagged it: the test delivers exactly two messages, and a third appearing due to a connector bug would pass right through. We changed it to `hasSize(2)`.

Same review caught that `@Timeout(5)` and `atMost(5, TimeUnit.SECONDS)` sitting together means JUnit can kill the thread before Awaitility gets to emit its failure message. We bumped eight tests to `@Timeout(10)` — including `sinkThrows_messageStillMarkedSeen_remainingDelivered`, which had a pre-existing `@Timeout(5)` with two `poll(3, SECONDS)` calls inside. That one was always potentially 6s.

For the `saveChanges()` call before `deliverDirect()` on multipart messages: without it, `appendMessage()` serialises before MIME boundaries are committed to the Content-Type header. SMTP calls `saveChanges()` implicitly. Direct injection doesn't.

Seventeen tests, all deterministic. I filed connectors#14 for the one holdout: `singlePlainTextMessage_deliveredWithCorrectFields` checks field mapping, not IDLE notification, and should be converted to `deliverDirect()` in the same pass as the others. Didn't do it now — the coverage argument for keeping it on SMTP is thin but the test isn't wrong as-is.
