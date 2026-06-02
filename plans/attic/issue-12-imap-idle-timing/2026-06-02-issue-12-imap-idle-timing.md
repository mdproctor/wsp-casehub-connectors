# IMAP IDLE Test Timing Fix (connectors#12) — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Eliminate IDLE notification timing flakiness in `EmailInboundConnectorTest` by switching content/field tests to `deliverDirect()` and replacing `receive()` with Awaitility.

**Architecture:** Five tests that check content parsing or field mapping are converted to pre-deliver via `deliverDirect()` + start-after (Path A — `processUnseen()` on first connect, deterministic). The four remaining SMTP-after-start tests keep SMTP because they specifically test the IMAP IDLE notification path; `receive()` is replaced with Awaitility(`atMost 5s`) so cold-JVM notification latency cannot cause a timeout. `@Timeout` is bumped where necessary to exceed the Awaitility window.

**Tech Stack:** Java 21, JUnit 5, GreenMail 2.1.3, Awaitility 4.3.0 (already on test classpath), AssertJ.

---

## Files

- **Modify only:** `email-inbound/src/test/java/io/casehub/connectors/email/inbound/EmailInboundConnectorTest.java`

No production code changes. No pom changes (Awaitility already in `email-inbound/pom.xml` test scope).

---

## Task 1: Add Awaitility import and replace `receive()`

The `receive()` helper is called by every test that reads a delivered message. Replacing it here applies the 5-second Awaitility window to all callers at once. `TimeUnit` is already imported — no change needed there.

**Files:**
- Modify: `email-inbound/src/test/java/io/casehub/connectors/email/inbound/EmailInboundConnectorTest.java`

- [ ] **Step 1.1: Add static Awaitility import**

In the import block, after the existing `import static org.assertj.core.api.Assertions.assertThat;` line, add:

```java
import static org.awaitility.Awaitility.await;
```

- [ ] **Step 1.2: Replace the `receive()` method**

Find:
```java
    /** Blocks until the IDLE loop delivers a message, fails after 3 s. */
    private InboundMessage receive() throws InterruptedException {
        final InboundMessage msg = captured.poll(3, TimeUnit.SECONDS);
        assertThat(msg).as("message not delivered within 3s — IDLE did not fire").isNotNull();
        return msg;
    }
```

Replace with:
```java
    /** Waits up to 5s for the IDLE loop or processUnseen to deliver a message. */
    private InboundMessage receive() {
        await().atMost(5, TimeUnit.SECONDS)
               .failMessage("message not delivered within 5s — IDLE did not fire or processUnseen missed it")
               .untilAsserted(() -> assertThat(captured).isNotEmpty());
        return captured.poll();
    }
```

Note: `captured.poll()` after `untilAsserted` is safe — there is exactly one writer (the connector's virtual thread) and one reader (the test thread). `untilAsserted` guarantees non-empty before `poll()` is called.

- [ ] **Step 1.3: Run the full test class to verify no regressions**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn -pl email-inbound test -Dtest=EmailInboundConnectorTest -q
```

Expected: `BUILD SUCCESS`. All 17 tests pass. (Some tests now have a 5s ceiling instead of 3s, but all should still complete well within that budget on a warm JVM.)

- [ ] **Step 1.4: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/connectors add email-inbound/src/test/java/io/casehub/connectors/email/inbound/EmailInboundConnectorTest.java
git -C /Users/mdproctor/claude/casehub/connectors commit -m "test(email-inbound): replace receive() with Awaitility 5s — tolerates IDLE notification latency

Refs #12"
```

---

## Task 2: Fix `multipleUnseenMessages_allDelivered` and bump @Timeout

The sequential `receive()` / `receive()` pattern had a hidden race: if IDLE fired between the two SMTP sends, the second message required a second notification cycle, each with its own 3s budget. Replace with a single atomic wait for both messages, then bump `@Timeout` to 10 (invariant: @Timeout must exceed atMost + buffer; 5s Awaitility + 5s buffer = 10s).

**Files:**
- Modify: `email-inbound/src/test/java/io/casehub/connectors/email/inbound/EmailInboundConnectorTest.java`

- [ ] **Step 2.1: Replace the two-phase receive with atomic wait**

Find:
```java
    @Test
    @Timeout(5)
    void multipleUnseenMessages_allDelivered() throws Exception {
        connector.start(captured::add);
        deliver("a@example.com", "First", "Body A");
        deliver("b@example.com", "Second", "Body B");

        final InboundMessage m1 = receive();
        final InboundMessage m2 = receive();
        assertThat(List.of(m1.content(), m2.content()))
                .containsExactlyInAnyOrder("Body A", "Body B");
    }
```

Replace with:
```java
    @Test
    @Timeout(10)
    void multipleUnseenMessages_allDelivered() throws Exception {
        connector.start(captured::add);
        deliver("a@example.com", "First", "Body A");
        deliver("b@example.com", "Second", "Body B");

        await().atMost(5, TimeUnit.SECONDS)
               .failMessage("both messages not delivered within 5s")
               .untilAsserted(() -> assertThat(captured).hasSizeGreaterThanOrEqualTo(2));
        final InboundMessage m1 = captured.poll();
        final InboundMessage m2 = captured.poll();
        assertThat(List.of(m1.content(), m2.content()))
                .containsExactlyInAnyOrder("Body A", "Body B");
    }
```

- [ ] **Step 2.2: Run the updated test to verify it passes**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn -pl email-inbound test -Dtest="EmailInboundConnectorTest#multipleUnseenMessages_allDelivered" -q
```

Expected: `BUILD SUCCESS`.

- [ ] **Step 2.3: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/connectors add email-inbound/src/test/java/io/casehub/connectors/email/inbound/EmailInboundConnectorTest.java
git -C /Users/mdproctor/claude/casehub/connectors commit -m "test(email-inbound): atomic wait for 2 messages in multipleUnseenMessages — removes delivery order race

Refs #12"
```

---

## Task 3: Bump @Timeout on remaining SMTP-after-start tests

`singlePlainTextMessage_deliveredWithCorrectFields` and `doubleStart_isNoOp` stay on SMTP-after-start (they test the IDLE notification path). With `receive()` now using `atMost(5, SECONDS)`, their `@Timeout(5)` would let JUnit kill the thread before Awaitility can emit its failure message. Bump to 10.

`messageMarkedSeen_notRedeliveredAfterRestart` is already `@Timeout(10)` — no change needed.

**Files:**
- Modify: `email-inbound/src/test/java/io/casehub/connectors/email/inbound/EmailInboundConnectorTest.java`

- [ ] **Step 3.1: Bump @Timeout on `singlePlainTextMessage_deliveredWithCorrectFields`**

Find:
```java
    @Test
    @Timeout(5)
    void singlePlainTextMessage_deliveredWithCorrectFields() throws Exception {
```

Replace:
```java
    @Test
    @Timeout(10)
    void singlePlainTextMessage_deliveredWithCorrectFields() throws Exception {
```

- [ ] **Step 3.2: Bump @Timeout on `doubleStart_isNoOp`**

Find:
```java
    @Test
    @Timeout(5)
    void doubleStart_isNoOp() throws Exception {
```

Replace:
```java
    @Test
    @Timeout(10)
    void doubleStart_isNoOp() throws Exception {
```

- [ ] **Step 3.3: Run those two tests to verify they pass**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn -pl email-inbound test -Dtest="EmailInboundConnectorTest#singlePlainTextMessage_deliveredWithCorrectFields+doubleStart_isNoOp" -q
```

Expected: `BUILD SUCCESS`.

- [ ] **Step 3.4: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/connectors add email-inbound/src/test/java/io/casehub/connectors/email/inbound/EmailInboundConnectorTest.java
git -C /Users/mdproctor/claude/casehub/connectors commit -m "test(email-inbound): bump @Timeout to 10s on IDLE notification tests — exceeds Awaitility atMost(5s)

Refs #12"
```

---

## Task 4: Convert content and attachment tests to `deliverDirect()`

These five tests are checking content parsing, metadata, or attachment handling — not IMAP IDLE notification behavior. Converting them to pre-deliver before `start()` puts them on Path A (`processUnseen()` on first connect), which is deterministic regardless of JVM startup speed. Each conversion is the same pattern: build MimeMessage → `deliverDirect(msg)` → `connector.start(captured::add)`.

For multipart messages (attachment tests), call `raw.saveChanges()` before `deliverDirect()` to commit the MIME boundary and Content-Type headers before GreenMail serializes the message for storage.

**Files:**
- Modify: `email-inbound/src/test/java/io/casehub/connectors/email/inbound/EmailInboundConnectorTest.java`

- [ ] **Step 4.1: Convert `htmlOnlyMessage_rawHtmlInContent`**

Find:
```java
    @Test
    @Timeout(5)
    void htmlOnlyMessage_rawHtmlInContent() throws Exception {
        connector.start(captured::add);
        final MimeMessage msg = new MimeMessage(Session.getInstance(new Properties()));
        msg.setFrom(new InternetAddress("sender@example.com"));
        msg.setRecipient(Message.RecipientType.TO, new InternetAddress("inbox@example.com"));
        msg.setSubject("HTML email");
        msg.setContent("<p>Rich content</p>", "text/html; charset=UTF-8");
        msg.setSentDate(Date.from(Instant.now()));
        deliverViaSMTP(msg);

        assertThat(receive().content()).isEqualTo("<p>Rich content</p>");
    }
```

Replace with:
```java
    @Test
    @Timeout(5)
    void htmlOnlyMessage_rawHtmlInContent() throws Exception {
        final MimeMessage msg = new MimeMessage(Session.getInstance(new Properties()));
        msg.setFrom(new InternetAddress("sender@example.com"));
        msg.setRecipient(Message.RecipientType.TO, new InternetAddress("inbox@example.com"));
        msg.setSubject("HTML email");
        msg.setContent("<p>Rich content</p>", "text/html; charset=UTF-8");
        msg.setSentDate(Date.from(Instant.now()));
        deliverDirect(msg);
        connector.start(captured::add);

        assertThat(receive().content()).isEqualTo("<p>Rich content</p>");
    }
```

- [ ] **Step 4.2: Convert `messageWithoutSubject_subjectKeyAbsent`**

Find:
```java
    @Test
    @Timeout(5)
    void messageWithoutSubject_subjectKeyAbsent() throws Exception {
        connector.start(captured::add);
        final MimeMessage msg = new MimeMessage(Session.getInstance(new Properties()));
        msg.setFrom(new InternetAddress("sender@example.com"));
        msg.setRecipient(Message.RecipientType.TO, new InternetAddress("inbox@example.com"));
        msg.setText("Body");
        msg.setSentDate(Date.from(Instant.now()));
        deliverViaSMTP(msg);

        assertThat(receive().metadata()).doesNotContainKey("subject");
    }
```

Replace with:
```java
    @Test
    @Timeout(5)
    void messageWithoutSubject_subjectKeyAbsent() throws Exception {
        final MimeMessage msg = new MimeMessage(Session.getInstance(new Properties()));
        msg.setFrom(new InternetAddress("sender@example.com"));
        msg.setRecipient(Message.RecipientType.TO, new InternetAddress("inbox@example.com"));
        msg.setText("Body");
        msg.setSentDate(Date.from(Instant.now()));
        deliverDirect(msg);
        connector.start(captured::add);

        assertThat(receive().metadata()).doesNotContainKey("subject");
    }
```

- [ ] **Step 4.3: Convert `messageWithNoAttachments_attachmentsEmptyAndCountIsZero`**

This test currently uses the `deliver()` convenience helper (SMTP). After conversion it builds the message inline for `deliverDirect()`.

Find:
```java
    @Test
    @Timeout(5)
    void messageWithNoAttachments_attachmentsEmptyAndCountIsZero() throws Exception {
        connector.start(captured::add);
        deliver("sender@example.com", "Plain", "Body");

        final InboundMessage msg = receive();
        assertThat(msg.attachments()).isEmpty();
        assertThat(msg.metadata()).containsEntry("attachment-count", "0");
    }
```

Replace with:
```java
    @Test
    @Timeout(5)
    void messageWithNoAttachments_attachmentsEmptyAndCountIsZero() throws Exception {
        final MimeMessage msg = new MimeMessage(Session.getInstance(new Properties()));
        msg.setFrom(new InternetAddress("sender@example.com"));
        msg.setRecipient(Message.RecipientType.TO, new InternetAddress("inbox@example.com"));
        msg.setSubject("Plain");
        msg.setText("Body");
        msg.setSentDate(Date.from(Instant.now()));
        deliverDirect(msg);
        connector.start(captured::add);

        final InboundMessage result = receive();
        assertThat(result.attachments()).isEmpty();
        assertThat(result.metadata()).containsEntry("attachment-count", "0");
    }
```

- [ ] **Step 4.4: Convert `messageWithPdfAttachment_attachmentDelivered`**

Find:
```java
    @Test
    @Timeout(5)
    void messageWithPdfAttachment_attachmentDelivered() throws Exception {
        connector.start(captured::add);

        final MimeMessage raw = new MimeMessage(Session.getInstance(new Properties()));
        raw.setFrom(new InternetAddress("sender@example.com"));
        raw.setRecipient(Message.RecipientType.TO, new InternetAddress("inbox@example.com"));
        raw.setSubject("Invoice");
        raw.setSentDate(Date.from(Instant.now()));

        final MimeMultipart multipart = new MimeMultipart();
        final MimeBodyPart textPart = new MimeBodyPart();
        textPart.setText("See attached");
        multipart.addBodyPart(textPart);
        final MimeBodyPart attPart = new MimeBodyPart();
        attPart.setContent(new byte[]{1, 2, 3}, "application/pdf");
        attPart.setDisposition(jakarta.mail.Part.ATTACHMENT);
        attPart.setFileName("invoice.pdf");
        multipart.addBodyPart(attPart);
        raw.setContent(multipart);

        deliverViaSMTP(raw);

        final InboundMessage msg = receive();
        assertThat(msg.content()).isEqualTo("See attached");
        assertThat(msg.attachments()).hasSize(1);
        assertThat(msg.attachments().get(0).filename()).isEqualTo("invoice.pdf");
        assertThat(msg.attachments().get(0).contentType()).isEqualTo("application/pdf");
        assertThat(msg.attachments().get(0).content()).isEqualTo(new byte[]{1, 2, 3});
        assertThat(msg.metadata()).containsEntry("attachment-count", "1");
    }
```

Replace with:
```java
    @Test
    @Timeout(5)
    void messageWithPdfAttachment_attachmentDelivered() throws Exception {
        final MimeMessage raw = new MimeMessage(Session.getInstance(new Properties()));
        raw.setFrom(new InternetAddress("sender@example.com"));
        raw.setRecipient(Message.RecipientType.TO, new InternetAddress("inbox@example.com"));
        raw.setSubject("Invoice");
        raw.setSentDate(Date.from(Instant.now()));

        final MimeMultipart multipart = new MimeMultipart();
        final MimeBodyPart textPart = new MimeBodyPart();
        textPart.setText("See attached");
        multipart.addBodyPart(textPart);
        final MimeBodyPart attPart = new MimeBodyPart();
        attPart.setContent(new byte[]{1, 2, 3}, "application/pdf");
        attPart.setDisposition(jakarta.mail.Part.ATTACHMENT);
        attPart.setFileName("invoice.pdf");
        multipart.addBodyPart(attPart);
        raw.setContent(multipart);
        raw.saveChanges();
        deliverDirect(raw);
        connector.start(captured::add);

        final InboundMessage msg = receive();
        assertThat(msg.content()).isEqualTo("See attached");
        assertThat(msg.attachments()).hasSize(1);
        assertThat(msg.attachments().get(0).filename()).isEqualTo("invoice.pdf");
        assertThat(msg.attachments().get(0).contentType()).isEqualTo("application/pdf");
        assertThat(msg.attachments().get(0).content()).isEqualTo(new byte[]{1, 2, 3});
        assertThat(msg.metadata()).containsEntry("attachment-count", "1");
    }
```

- [ ] **Step 4.5: Convert `messageWithMultipleAttachments_allCollected`**

Find:
```java
    @Test
    @Timeout(5)
    void messageWithMultipleAttachments_allCollected() throws Exception {
        connector.start(captured::add);

        final MimeMessage raw = new MimeMessage(Session.getInstance(new Properties()));
        raw.setFrom(new InternetAddress("sender@example.com"));
        raw.setRecipient(Message.RecipientType.TO, new InternetAddress("inbox@example.com"));
        raw.setSubject("Files");
        raw.setSentDate(Date.from(Instant.now()));

        final MimeMultipart multipart = new MimeMultipart();
        final MimeBodyPart textPart = new MimeBodyPart();
        textPart.setText("Two files");
        multipart.addBodyPart(textPart);

        final MimeBodyPart pdf = new MimeBodyPart();
        pdf.setContent(new byte[]{1}, "application/pdf");
        pdf.setDisposition(jakarta.mail.Part.ATTACHMENT);
        pdf.setFileName("a.pdf");
        multipart.addBodyPart(pdf);

        final MimeBodyPart img = new MimeBodyPart();
        img.setContent(new byte[]{2, 3}, "image/png");
        img.setDisposition(jakarta.mail.Part.ATTACHMENT);
        img.setFileName("b.png");
        multipart.addBodyPart(img);

        raw.setContent(multipart);
        deliverViaSMTP(raw);

        final InboundMessage msg = receive();
        assertThat(msg.attachments()).hasSize(2);
        assertThat(msg.attachments()).extracting(Attachment::filename)
                .containsExactlyInAnyOrder("a.pdf", "b.png");
        assertThat(msg.metadata()).containsEntry("attachment-count", "2");
    }
```

Replace with:
```java
    @Test
    @Timeout(5)
    void messageWithMultipleAttachments_allCollected() throws Exception {
        final MimeMessage raw = new MimeMessage(Session.getInstance(new Properties()));
        raw.setFrom(new InternetAddress("sender@example.com"));
        raw.setRecipient(Message.RecipientType.TO, new InternetAddress("inbox@example.com"));
        raw.setSubject("Files");
        raw.setSentDate(Date.from(Instant.now()));

        final MimeMultipart multipart = new MimeMultipart();
        final MimeBodyPart textPart = new MimeBodyPart();
        textPart.setText("Two files");
        multipart.addBodyPart(textPart);

        final MimeBodyPart pdf = new MimeBodyPart();
        pdf.setContent(new byte[]{1}, "application/pdf");
        pdf.setDisposition(jakarta.mail.Part.ATTACHMENT);
        pdf.setFileName("a.pdf");
        multipart.addBodyPart(pdf);

        final MimeBodyPart img = new MimeBodyPart();
        img.setContent(new byte[]{2, 3}, "image/png");
        img.setDisposition(jakarta.mail.Part.ATTACHMENT);
        img.setFileName("b.png");
        multipart.addBodyPart(img);

        raw.setContent(multipart);
        raw.saveChanges();
        deliverDirect(raw);
        connector.start(captured::add);

        final InboundMessage msg = receive();
        assertThat(msg.attachments()).hasSize(2);
        assertThat(msg.attachments()).extracting(Attachment::filename)
                .containsExactlyInAnyOrder("a.pdf", "b.png");
        assertThat(msg.metadata()).containsEntry("attachment-count", "2");
    }
```

- [ ] **Step 4.6: Run the five converted tests to verify they pass**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn -pl email-inbound test -Dtest="EmailInboundConnectorTest#htmlOnlyMessage_rawHtmlInContent+messageWithoutSubject_subjectKeyAbsent+messageWithNoAttachments_attachmentsEmptyAndCountIsZero+messageWithPdfAttachment_attachmentDelivered+messageWithMultipleAttachments_allCollected" -q
```

Expected: `BUILD SUCCESS`. If an attachment test fails asserting content bytes, check `raw.saveChanges()` is called before `deliverDirect(raw)`.

- [ ] **Step 4.7: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/connectors add email-inbound/src/test/java/io/casehub/connectors/email/inbound/EmailInboundConnectorTest.java
git -C /Users/mdproctor/claude/casehub/connectors commit -m "test(email-inbound): convert content/field tests to deliverDirect() — eliminates IDLE race for 5 tests

These tests check parsing, not IDLE notification. Pre-delivering via
MailFolder.appendMessage() puts them on processUnseen() path which is
deterministic regardless of JVM startup speed.

Closes #12"
```

---

## Task 5: Full verification run

- [ ] **Step 5.1: Run the complete test suite for the email-inbound module**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn -pl email-inbound test
```

Expected: `BUILD SUCCESS`, 17 tests pass, 0 failures, 0 errors. No `@Timeout` kills. If any test fails:
- A `ConditionTimeoutException` from Awaitility → the IDLE notification took > 5s; retry once to confirm it's truly flaky, not just an outlier
- An assertion error in a converted test → check `deliverDirect()` was called before `connector.start()` and that `saveChanges()` precedes `deliverDirect()` for multipart messages

- [ ] **Step 5.2: Run the full project build**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn clean install
```

Expected: `BUILD SUCCESS` across all modules.

---

## Self-Review

**Spec coverage:**
- ✅ `receive()` replaced with Awaitility → Task 1
- ✅ `multipleUnseenMessages_allDelivered` atomic wait → Task 2
- ✅ `@Timeout` bumps (doubleStart, singlePlainText) → Task 3
- ✅ `messageMarkedSeen_notRedeliveredAfterRestart` already at @Timeout(10), receive() update applied by Task 1 → covered
- ✅ Five content tests converted to `deliverDirect()` → Task 4
- ✅ `raw.saveChanges()` before `deliverDirect()` for multipart → Tasks 4.4 and 4.5
- ✅ Import added → Task 1

**Placeholder scan:** No TBDs. All code blocks show complete method bodies.

**Type consistency:** `captured` is `LinkedBlockingQueue<InboundMessage>` throughout. `receive()` returns `InboundMessage`. `await()` is `org.awaitility.Awaitility.await`. All references consistent.
