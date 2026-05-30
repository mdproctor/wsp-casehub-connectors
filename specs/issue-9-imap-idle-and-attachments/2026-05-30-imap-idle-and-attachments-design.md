# Design Spec — IMAP IDLE + Attachment Support

**Issues:** casehubio/connectors#9 (IMAP IDLE), casehubio/connectors#10 (attachment support)  
**Branch:** `issue-9-imap-idle-and-attachments`  
**Date:** 2026-05-30

---

## Context

`EmailInboundConnector` was shipped in connectors#7 as a polling-based IMAP connector: one
`ScheduledExecutorService` per account, opening a fresh connection every `pollIntervalSeconds`,
searching UNSEEN messages, delivering via `InboundMessageSink`, and closing. Two capabilities
were deferred: near-real-time delivery (IDLE) and binary/attachment support.

This spec covers both in one branch. They compose cleanly: IDLE replaces the execution model,
attachments extend the data model. Neither depends on the other.

---

## Issue #9 — IMAP IDLE

### Decision

IMAP IDLE (RFC 2177, 1997) replaces polling entirely. No polling fallback. Every target
deployment (Gmail, Outlook 365, Fastmail, Dovecot, Exchange, ProtonMail) has supported IDLE
for 20+ years. A fallback code path would add state management, failure-categorisation
complexity, and divergence risk — for a server capability that doesn't exist in the target
deployment universe.

When IDLE fails persistently (wrong server, firewall, credentials), the reconnect loop logs
at `SEVERE` after 5+ consecutive failures — loud, actionable, no silent degradation.

### Execution Model

One virtual thread per account, started at `InboundConnector.start()`, running a blocking
IDLE loop. Virtual threads are correct for this: each thread blocks on a socket for up to
29 minutes at a time. Platform threads would hold an OS thread indefinitely per account;
virtual threads park during the block at zero OS-thread cost.

```
start():
  for each account:
    Thread.ofVirtual().name("email-inbound-<id>-idle").start(() -> idleLoop(account, sink))

idleLoop(account, sink):
  backoffSeconds = 1
  consecutiveFailures = 0
  while not stopping:
    try:
      store = connect(account)
      openStores.add(store)
      folder = store.getFolder(account.folder()) as IMAPFolder
      folder.open(READ_WRITE)
      backoffSeconds = 1; consecutiveFailures = 0
      log INFO "IDLE connected for account <id>"
      while not stopping:
        folder.idle(false)          // blocks until server notification or timeout
        processUnseen(folder, account, sink)
    catch Exception e:
      openStores.remove(store)
      if not stopping:
        consecutiveFailures++
        level = consecutiveFailures >= 5 ? SEVERE : WARNING
        log(level, "connection failed for account <id> (attempt N): " + e.getMessage())
        sleep(backoffSeconds * 1000)
        backoffSeconds = min(backoffSeconds * 2, account.reconnectDelaySeconds())
    finally:
      openStores.remove(store)
      closeQuietly(folder, store)
```

**`folder.idle(false)`** blocks until the IMAP server sends any notification (new message,
flag change, expunge, server-side keepalive, timeout). After it returns, `processUnseen()`
runs the same UNSEEN search + SEEN flag logic as the original `pollAccount()`. At-least-once
delivery guarantee is preserved: if shutdown interrupts between `sink.receive()` and the SEEN
flag write, the message redelivers on next startup. Observers must be idempotent.

**Reconnect strategy:** exponential backoff (1s → 2s → 4s → ... capped at
`reconnectDelaySeconds`). Resets to 1s on successful connection. Never gives up — transient
failures (server restarts, network blips) self-heal; permanent failures produce SEVERE logs.

### Stop Mechanism

```
stop():
  stopping = true
  copy(openStores).forEach(store -> store.close() quietly)   // causes idle() to throw FolderClosedException
  idleThreads.forEach(Thread::interrupt)                      // unblocks any thread in sleepQuietly()
```

`store.close()` closes the underlying socket, causing `folder.idle(false)` to throw
`FolderClosedException`. The inner `while (!stopping)` exits. The outer loop sees `stopping =
true` and exits cleanly. `Thread.interrupt()` handles threads blocked in reconnect backoff
sleep. `openStores` is `CopyOnWriteArrayList` — safe concurrent iteration from the Quarkus
shutdown thread while IDLE loops run on virtual threads.

### `EmailInboundAccount` Change

`pollIntervalSeconds` renamed to `reconnectDelaySeconds`. The field now caps the exponential
reconnect backoff rather than driving a poll schedule. The name is honest about what it controls.

```java
public record EmailInboundAccount(
    String id, String host, int port, boolean tls,
    String username, String password, String folder,
    int reconnectDelaySeconds)   // was: pollIntervalSeconds
```

### Configuration Change

MP Config property renamed:

| Old | New | Default |
|-----|-----|---------|
| `casehub.connectors.email-inbound.poll-interval-seconds` | `casehub.connectors.email-inbound.reconnect-delay-seconds` | `60` |

`GreenMailResource` drops `poll-interval-seconds=1` entirely. IDLE delivers immediately on
message arrival; the reconnect delay is irrelevant to test timing.

---

## Issue #10 — Attachment Support

### `Attachment` Record — `core` module

```java
public record Attachment(String filename, String contentType, byte[] content) {
    public Attachment {
        content = content == null ? new byte[0] : content.clone();
    }
    @Override public byte[] content() { return content.clone(); }
}
```

`filename` is nullable — MIME does not require a filename. `content` is defensively copied
on construction and access — `byte[]` is mutable; records must enforce their own immutability
for library use. `contentType` carries the base MIME type only (parameters stripped).

Lives in `core` because `InboundMessage` (also in `core`) references it. Any connector that
eventually supports attachments uses the same type — no duplication.

### `InboundMessage` Change — `core` module

`List<Attachment> attachments` added between `content` and `receivedAt`:

```java
public record InboundMessage(
    String connectorId,
    String externalSenderId,
    String externalChannelRef,
    String content,
    List<Attachment> attachments,   // NEW
    Instant receivedAt,
    Map<String, String> metadata)
```

Two convenience constructors delegate to the canonical, covering all existing call sites:

```java
// no attachments, with metadata — all webhook connector call sites compile unchanged
public InboundMessage(String connectorId, String externalSenderId, String externalChannelRef,
                      String content, Instant receivedAt, Map<String, String> metadata) {
    this(connectorId, externalSenderId, externalChannelRef, content, List.of(), receivedAt, metadata);
}

// no attachments, no metadata
public InboundMessage(String connectorId, String externalSenderId, String externalChannelRef,
                      String content, Instant receivedAt) {
    this(connectorId, externalSenderId, externalChannelRef, content, List.of(), receivedAt, Map.of());
}
```

`attachments` is always `List.of()` for connectors that produce no attachments — never null.
The javadoc comment "Text content only in v1" is removed.

### `ContentExtractor` Refactor — `email-inbound` module

**`ExtractionResult`** — package-private record:
```java
record ExtractionResult(String content, List<Attachment> attachments) {}
```

**Single MIME-tree pass** replacing the previous two-pass structure (`extractText()` then
`extractHtml()`). One traversal, one result:

```
extract(Part part):
  acc = new Accumulator()
  traverse(part, acc)
  return ExtractionResult(acc.resolveContent(), acc.attachments)

traverse(Part part, Accumulator acc):
  if part is text/plain  → acc.plainText = content (if not already set)
  if part is text/html   → acc.htmlText  = content (if not already set)
  if part is multipart/* → recurse into each body part
  else                   → acc.attachments.add(toAttachment(part))

Accumulator.resolveContent():
  plainText ?? htmlText ?? ""
```

**What counts as an attachment:** any MIME part that is not `text/plain`, not `text/html`,
and not `multipart/*`. Content-Disposition (inline vs attachment) is not consulted —
the MIME type drives the decision. This includes inline images in HTML newsletters, embedded
audio, and any unknown content type. Observers that want only explicit attachments filter
themselves; the connector delivers everything the message contains.

**`toAttachment(Part)`:** reads `Part.getFileName()` (null if absent), strips MIME type
parameters for `contentType`, reads `Part.getInputStream().readAllBytes()` for `content`.

### `buildMetadata()` — `attachment-count` key

When attachments are present, `buildMetadata()` adds:

```
"attachment-count" → String.valueOf(attachments.size())
```

Allows CDI observers to branch on attachment presence without accessing binary data:
`message.metadata().containsKey("attachment-count")`.

### `EmailInboundConnector.toInboundMessage()` — updated

```java
final ExtractionResult extracted = ContentExtractor.extract(msg);
return new InboundMessage(
    ID,
    extractSenderId(msg),
    extractChannelRef(msg, account),
    extracted.content(),
    extracted.attachments(),
    resolveReceivedAt(msg),
    buildMetadata(account, msg, extracted.attachments().size()));
```

---

## Testing Strategy

### `EmailInboundConnectorTest` (unit, GreenMailExtension)

`captured` changes from `List<InboundMessage>` to `LinkedBlockingQueue<InboundMessage>`.
Tests start the connector, deliver via SMTP or direct GreenMail inject, then call
`captured.poll(3, SECONDS)` to block until IDLE delivers.

Helper:
```java
private InboundMessage receive() throws InterruptedException {
    InboundMessage msg = captured.poll(3, TimeUnit.SECONDS);
    assertThat(msg).as("message not delivered within 3s").isNotNull();
    return msg;
}
```

Tests formerly calling `pollAccount()` directly become start→deliver→receive assertions.

At-least-once (SEEN flag) tests: deliver → receive → `stop()` → `start()` new connector →
assert nothing received. More realistic than the old direct-call approach.

IDLE reconnect: deliver to a good account after a failed connection attempt — assert eventual
delivery after reconnect.

New attachment tests:
- `messageWithPdfAttachment_attachmentInResult` — filename, contentType, content, metadata key
- `attachmentOnlyMessage_emptyContent_attachmentPresent`
- `multipleAttachments_allCollected`
- `inlineImage_collectedAsAttachment`
- `messageWithNoAttachments_attachmentsEmpty` (regression guard)

### `ContentExtractorTest` (unit, no IMAP)

New cases:
- `multipartMixed_textAndPdf_textInContent_pdfInAttachments`
- `multipartMixed_noText_attachmentsPresent_contentEmpty`
- `inlineImage_collectedAsAttachment`
- `attachmentWithNoFilename_filenameIsNull`
- `contentTypeParametersStripped_baseTypeOnly`

Existing text/html/multipart/alternative cases retained.

### `EmailInboundConnectorQuarkusTest` (Quarkus CDI integration)

`GreenMailResource` drops `poll-interval-seconds=1`. Test body unchanged — 2-second
`capture.poll()` wait is ample for IDLE delivery.

### Unchanged

- `InboundConnectorServiceTest` — mocks `InboundConnector`; constructs `InboundMessage`
  via 6-arg convenience constructor, which still compiles.
- All webhook inbound tests — same convenience constructor, no attachments involved.
- `DefaultEmailInboundAccountProviderTest` — update config property key assertions only.

---

## Files Changed

| File | Change |
|------|--------|
| `core/.../Attachment.java` | New record |
| `core/.../InboundMessage.java` | Add `attachments` field + convenience constructors |
| `email-inbound/.../EmailInboundAccount.java` | Rename `pollIntervalSeconds` → `reconnectDelaySeconds` |
| `email-inbound/.../EmailInboundConnector.java` | Full rewrite: IDLE loop, virtual threads, stop |
| `email-inbound/.../DefaultEmailInboundAccountProvider.java` | Rename config property |
| `email-inbound/.../ContentExtractor.java` | Single-pass refactor returning `ExtractionResult` |
| `email-inbound/.../EmailInboundConnectorTest.java` | IDLE test model + attachment tests |
| `email-inbound/.../EmailInboundConnectorQuarkusTest.java` | Drop poll-interval config |
| `email-inbound/.../GreenMailResource.java` | Drop poll-interval config |
| `email-inbound/.../DefaultEmailInboundAccountProviderTest.java` | Update property key |

**Not changed:** webhook module (convenience constructors cover all call sites),
`InboundConnectorService`, `EmailInboundAccountProvider` SPI.

---

## Platform Coherence

- `inbound-connector-id-is-type-not-account`: `connectorId` remains `"email-inbound"` (type). ✓
- `inbound-connector-type-separation`: `EmailInboundConnector` implements `InboundConnector` (pull). ✓
- Module tier structure: `Attachment` in pure-Java `core`; IMAP logic in `email-inbound`. ✓
- No Flyway, no JPA, no new dependencies. ✓
- `casehub-connectors` remains dependency-free within the casehubio ecosystem. ✓
