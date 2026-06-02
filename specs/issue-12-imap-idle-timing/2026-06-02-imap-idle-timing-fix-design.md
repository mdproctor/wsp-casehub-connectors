# Design: Fix IMAP IDLE Test Timing Flakiness (connectors#12)

## Problem

`EmailInboundConnectorTest` has two intermittently failing tests under Maven cold-build conditions:

- `messageWithNoAttachments_attachmentsEmptyAndCountIsZero`
- `multipleUnseenMessages_allDelivered`

**Root cause — two execution paths exist after `connector.start()`:**

`start()` returns immediately after submitting a virtual thread. That thread races through:
IMAP connect → `folder.open` → `processUnseen()` → `folder.idle(true)` (blocks).

The test thread delivers a message via SMTP immediately after `start()`.

- **Path A (reliable):** SMTP completes before virtual thread connects → `processUnseen()` on first
  connect finds the message deterministically.
- **Path B (flaky):** Virtual thread enters `idle(true)` before SMTP completes → depends on
  GreenMail firing an EXISTS notification. Notification latency is non-deterministic under JVM
  startup load. The current 3-second `BlockingQueue.poll()` timeout can be exceeded on a cold JVM.

`multipleUnseenMessages_allDelivered` has a further structural issue: two sequential `receive()` calls
each with independent 3-second waits. If the connector processes both messages in separate IDLE cycles,
the second `receive()` waits for a second notification, compounding latency.

## Scope

Test-only changes. No production code, no SPI changes, no new dependencies (Awaitility is already
on the test classpath in `email-inbound/pom.xml`).

## Fix

### 1. Replace `receive()` with Awaitility

```java
// Before
private InboundMessage receive() throws InterruptedException {
    final InboundMessage msg = captured.poll(3, TimeUnit.SECONDS);
    assertThat(msg).as("message not delivered within 3s — IDLE did not fire").isNotNull();
    return msg;
}

// After
private InboundMessage receive() {
    await().atMost(5, SECONDS)
           .failMessage("message not delivered within 5s — IDLE did not fire or processUnseen missed it")
           .until(() -> !captured.isEmpty());
    return captured.poll();
}
```

Awaitility polls every 100ms adaptively. Failure message is explicit without a separate assertion.
No JTA/CDI context issue — this is a plain JUnit test, not `@QuarkusTest` (see GE-20260519-e193d2).

### 2. Atomic wait in `multipleUnseenMessages_allDelivered`

```java
// Before
final InboundMessage m1 = receive();
final InboundMessage m2 = receive();

// After
await().atMost(5, SECONDS)
       .failMessage("both messages not delivered within 5s")
       .until(() -> captured.size() >= 2);
final InboundMessage m1 = captured.poll();
final InboundMessage m2 = captured.poll();
```

Eliminates sequential two-phase waiting. Waits for both messages to be in the queue
before pulling either — correct regardless of whether they arrived in one or two IDLE cycles.

### 3. `@Timeout` bumps — scoped to SMTP-after-start tests only

Tests that deliver via SMTP *after* `start()` go through Path B and need `@Timeout(10)`:

- `doubleStart_isNoOp`
- `singlePlainTextMessage_deliveredWithCorrectFields`
- `multipleUnseenMessages_allDelivered`
- `htmlOnlyMessage_rawHtmlInContent`
- `messageWithoutSubject_subjectKeyAbsent`
- `messageWithPdfAttachment_attachmentDelivered`
- `messageWithNoAttachments_attachmentsEmptyAndCountIsZero`
- `messageWithMultipleAttachments_allCollected`

Tests that pre-deliver before `start()` are on Path A (reliable) — unchanged:
`missingFromHeader_...`, `missingToHeader_...`, `sinkThrows_...`.

### 4. Leave unchanged

`captured.poll(500, TimeUnit.MILLISECONDS)` in `doubleStart_isNoOp` — this is a negative
assertion (nothing arrives). Fixed wait is correct here; Awaitility would not help.

## What this does not fix

The IDLE notification latency is in GreenMail internals. This fix makes the tests resilient
to that latency; it does not change GreenMail's behaviour. If the notification takes > 5s, tests
still fail — but that would indicate a more serious GreenMail or JVM issue worth investigating.

## References

- GE-20260501-0586a4: Awaitility `during()` for stable-count assertions
- GE-20260515-ed10ee: `untilAsserted` vs `during` for exact async counts
- GE-20260519-e193d2: Awaitility + JTA context caveat (does not apply here — plain JUnit)
