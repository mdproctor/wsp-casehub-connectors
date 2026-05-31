# IMAP IDLE + Attachment Support Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Replace per-account IMAP polling with IDLE-based near-real-time delivery (#9), then add binary attachment support to `InboundMessage` via a single-pass MIME extractor (#10).

**Architecture:** Phase 1 replaces `ScheduledExecutorService` with a `VirtualThreadPerTaskExecutor` running one blocking IDLE loop per account; exponential backoff reconnects on failure; `volatile boolean stopping` + `store.close()` drives shutdown. Phase 2 adds an `Attachment` record to `core`, extends `InboundMessage` with `List<Attachment>`, and refactors `ContentExtractor` to a single-pass MIME traversal returning `ExtractionResult(content, attachments)`.

**Tech Stack:** Java 21 (virtual threads), Quarkus 3.32.2, angus-mail 2.0.3 (`org.eclipse.angus.mail.imap.IMAPFolder`), `jakarta.mail.FolderClosedException` / `StoreClosedException` (API jar), GreenMail 2.x, JUnit 5, AssertJ.

**Build command:** `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn clean install`
**Per-module test:** `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl <module>`

---

## File Map

**`core` module**
- `core/src/main/java/io/casehub/connectors/Attachment.java` — NEW
- `core/src/main/java/io/casehub/connectors/InboundMessage.java` — ADD field + constructors

**`email-inbound` module**
- `email-inbound/src/main/java/io/casehub/connectors/email/inbound/EmailInboundAccount.java` — RENAME field
- `email-inbound/src/main/java/io/casehub/connectors/email/inbound/DefaultEmailInboundAccountProvider.java` — RENAME field + config key
- `email-inbound/src/main/java/io/casehub/connectors/email/inbound/EmailInboundConnector.java` — FULL REWRITE (IDLE) + UPDATE (toInboundMessage, buildMetadata)
- `email-inbound/src/main/java/io/casehub/connectors/email/inbound/ExtractionResult.java` — NEW (own file, package-private)
- `email-inbound/src/main/java/io/casehub/connectors/email/inbound/ContentExtractor.java` — REFACTOR (single-pass with Accumulator)

**`email-inbound` tests**
- `email-inbound/src/test/java/io/casehub/connectors/email/inbound/EmailInboundConnectorTest.java` — FULL REWRITE (IDLE model + attachments)
- `email-inbound/src/test/java/io/casehub/connectors/email/inbound/ContentExtractorTest.java` — FULL REWRITE (ExtractionResult API + attachment cases)
- `email-inbound/src/test/java/io/casehub/connectors/email/inbound/DefaultEmailInboundAccountProviderTest.java` — RENAME field refs
- `email-inbound/src/test/java/io/casehub/connectors/email/inbound/GreenMailResource.java` — DROP poll-interval key
- `email-inbound/src/test/java/io/casehub/connectors/email/inbound/EmailInboundConnectorQuarkusTest.java` — ADD attachment assertions

---

## Phase 1 — Issue #9: IMAP IDLE

---

### Task 1: Rename `pollIntervalSeconds` → `reconnectDelaySeconds`

**Files:**
- Modify: `email-inbound/src/main/java/io/casehub/connectors/email/inbound/EmailInboundAccount.java`
- Modify: `email-inbound/src/main/java/io/casehub/connectors/email/inbound/DefaultEmailInboundAccountProvider.java`
- Modify: `email-inbound/src/test/java/io/casehub/connectors/email/inbound/DefaultEmailInboundAccountProviderTest.java`

- [ ] **Step 1: Replace `EmailInboundAccount.java`**

```java
package io.casehub.connectors.email.inbound;

/**
 * Configuration for one IMAP account to monitor via IDLE.
 *
 * <p>{@code id} appears in {@code InboundMessage.metadata["account-id"]}
 * (not in {@code connectorId} — that is always {@code "email-inbound"}).
 *
 * <p>{@code reconnectDelaySeconds} caps the exponential backoff applied between
 * connection attempts when the IMAP IDLE connection drops.
 */
public record EmailInboundAccount(
        String id,
        String host,
        int port,
        boolean tls,
        String username,
        String password,
        String folder,
        int reconnectDelaySeconds) {
}
```

- [ ] **Step 2: Replace `DefaultEmailInboundAccountProvider.java`**

```java
package io.casehub.connectors.email.inbound;

import java.util.List;

import jakarta.enterprise.context.ApplicationScoped;

import io.quarkus.arc.DefaultBean;
import org.eclipse.microprofile.config.inject.ConfigProperty;

/**
 * Default {@link EmailInboundAccountProvider} — reads a single IMAP account from
 * MP Config. Returns an empty list when {@code host} is blank (connector is inactive).
 *
 * <p>Override by providing an {@code @ApplicationScoped} bean without {@code @DefaultBean}.
 */
@DefaultBean
@ApplicationScoped
public class DefaultEmailInboundAccountProvider implements EmailInboundAccountProvider {

    @ConfigProperty(name = "casehub.connectors.email-inbound.host", defaultValue = "")
    String host;

    @ConfigProperty(name = "casehub.connectors.email-inbound.port", defaultValue = "993")
    int port;

    @ConfigProperty(name = "casehub.connectors.email-inbound.tls", defaultValue = "true")
    boolean tls;

    @ConfigProperty(name = "casehub.connectors.email-inbound.username", defaultValue = "")
    String username;

    @ConfigProperty(name = "casehub.connectors.email-inbound.password", defaultValue = "")
    String password;

    @ConfigProperty(name = "casehub.connectors.email-inbound.folder", defaultValue = "INBOX")
    String folder;

    @ConfigProperty(name = "casehub.connectors.email-inbound.reconnect-delay-seconds", defaultValue = "60")
    int reconnectDelaySeconds;

    DefaultEmailInboundAccountProvider() {}

    DefaultEmailInboundAccountProvider(final String host, final int port, final boolean tls,
                                       final String username, final String password,
                                       final String folder, final int reconnectDelaySeconds) {
        this.host = host;
        this.port = port;
        this.tls = tls;
        this.username = username;
        this.password = password;
        this.folder = folder;
        this.reconnectDelaySeconds = reconnectDelaySeconds;
    }

    @Override
    public List<EmailInboundAccount> accounts() {
        if (host == null || host.isBlank()) {
            return List.of();
        }
        return List.of(new EmailInboundAccount(
                EmailInboundConnector.ID, host, port, tls, username, password,
                folder, reconnectDelaySeconds));
    }
}
```

- [ ] **Step 3: Replace `DefaultEmailInboundAccountProviderTest.java`**

```java
package io.casehub.connectors.email.inbound;

import static org.assertj.core.api.Assertions.assertThat;

import java.util.List;

import org.junit.jupiter.api.Test;

class DefaultEmailInboundAccountProviderTest {

    @Test
    void blankHost_returnsEmptyList() {
        final DefaultEmailInboundAccountProvider provider =
                new DefaultEmailInboundAccountProvider("", 993, true, "user", "pass", "INBOX", 60);
        assertThat(provider.accounts()).isEmpty();
    }

    @Test
    void nullHost_returnsEmptyList() {
        final DefaultEmailInboundAccountProvider provider =
                new DefaultEmailInboundAccountProvider(null, 993, true, "user", "pass", "INBOX", 60);
        assertThat(provider.accounts()).isEmpty();
    }

    @Test
    void configuredHost_returnsSingleAccount() {
        final DefaultEmailInboundAccountProvider provider =
                new DefaultEmailInboundAccountProvider(
                        "imap.example.com", 993, true, "user@example.com",
                        "secret", "INBOX", 60);

        final List<EmailInboundAccount> accounts = provider.accounts();
        assertThat(accounts).hasSize(1);

        final EmailInboundAccount account = accounts.get(0);
        assertThat(account.id()).isEqualTo(EmailInboundConnector.ID);
        assertThat(account.host()).isEqualTo("imap.example.com");
        assertThat(account.port()).isEqualTo(993);
        assertThat(account.tls()).isTrue();
        assertThat(account.username()).isEqualTo("user@example.com");
        assertThat(account.password()).isEqualTo("secret");
        assertThat(account.folder()).isEqualTo("INBOX");
        assertThat(account.reconnectDelaySeconds()).isEqualTo(60);
    }

    @Test
    void customFolder_preservedInAccount() {
        final DefaultEmailInboundAccountProvider provider =
                new DefaultEmailInboundAccountProvider(
                        "imap.example.com", 143, false, "user", "pass", "Support", 30);
        assertThat(provider.accounts().get(0).folder()).isEqualTo("Support");
        assertThat(provider.accounts().get(0).tls()).isFalse();
        assertThat(provider.accounts().get(0).reconnectDelaySeconds()).isEqualTo(30);
    }
}
```

- [ ] **Step 4: Run tests to verify rename compiles cleanly**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl email-inbound -Dtest=DefaultEmailInboundAccountProviderTest -q
```

Expected: BUILD SUCCESS

- [ ] **Step 5: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/connectors add \
  email-inbound/src/main/java/io/casehub/connectors/email/inbound/EmailInboundAccount.java \
  email-inbound/src/main/java/io/casehub/connectors/email/inbound/DefaultEmailInboundAccountProvider.java \
  email-inbound/src/test/java/io/casehub/connectors/email/inbound/DefaultEmailInboundAccountProviderTest.java
git -C /Users/mdproctor/claude/casehub/connectors commit -m "refactor(email-inbound): rename pollIntervalSeconds → reconnectDelaySeconds — Refs #9"
```

---

### Task 2: Rewrite `EmailInboundConnector` for IMAP IDLE

**Files:**
- Modify: `email-inbound/src/test/java/io/casehub/connectors/email/inbound/EmailInboundConnectorTest.java`
- Modify: `email-inbound/src/main/java/io/casehub/connectors/email/inbound/EmailInboundConnector.java`
- Modify: `email-inbound/src/test/java/io/casehub/connectors/email/inbound/GreenMailResource.java`

**Key design invariants the implementation must honour:**

- `stopping` must be `volatile` — written by Quarkus shutdown thread, read by virtual threads
- `executor` null-check is the double-start guard (`if (executor != null) return`)
- Race guard: after `openStores.add(store)`, immediately check `if (stopping)` — `stop()` may have already iterated `openStores`
- `processUnseen()` is called BEFORE the first `folder.idle(false)` — catches messages already in the mailbox when the connector starts
- `FolderClosedException` / `StoreClosedException` caught separately at INFO, no backoff, no `consecutiveFailures++` — these cover normal server IDLE timeouts. A comment explains that consecutive quick disconnects eventually fail reconnect and hit the generic backoff path
- `openStores.remove(store)` only in `finally` — both catch blocks do NOT call `remove(store)`
- `buildProperties()` adds `mail.imap.timeout=300000` / `mail.imaps.timeout=300000` (5 min socket timeout)
- `buildMetadata()` now takes `int attachmentCount` and always writes `"attachment-count"` key (value `"0"` until Phase 2 completes)

- [ ] **Step 1: Write the failing tests first — replace `EmailInboundConnectorTest.java`**

SMTP-delivery tests start the connector first then deliver (IDLE fires on SMTP notification).
Direct-delivery tests (`deliverDirect`) pre-deliver before `start()` so `processUnseen()` on first connect catches them — avoids relying on whether GreenMail sends IDLE notifications for direct appends.

```java
package io.casehub.connectors.email.inbound;

import static org.assertj.core.api.Assertions.assertThat;

import java.time.Instant;
import java.util.Date;
import java.util.List;
import java.util.Map;
import java.util.Properties;
import java.util.concurrent.LinkedBlockingQueue;
import java.util.concurrent.TimeUnit;

import jakarta.mail.Message;
import jakarta.mail.Session;
import jakarta.mail.Transport;
import jakarta.mail.internet.InternetAddress;
import jakarta.mail.internet.MimeMessage;

import com.icegreen.greenmail.configuration.GreenMailConfiguration;
import com.icegreen.greenmail.junit5.GreenMailExtension;
import com.icegreen.greenmail.store.MailFolder;
import com.icegreen.greenmail.user.GreenMailUser;
import com.icegreen.greenmail.util.ServerSetup;
import io.casehub.connectors.InboundMessage;

import org.junit.jupiter.api.AfterEach;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;
import org.junit.jupiter.api.Timeout;
import org.junit.jupiter.api.extension.RegisterExtension;

class EmailInboundConnectorTest {

    // Port 0 = OS-assigned, avoids conflict with GreenMailResource (fixed ports for @QuarkusTest)
    @RegisterExtension
    static final GreenMailExtension GREEN_MAIL = new GreenMailExtension(new ServerSetup[]{
            new ServerSetup(0, "localhost", ServerSetup.PROTOCOL_SMTP),
            new ServerSetup(0, "localhost", ServerSetup.PROTOCOL_IMAP)})
            .withConfiguration(GreenMailConfiguration.aConfig()
                    .withUser("inbox@example.com", "password"))
            .withPerMethodLifecycle(false);

    private LinkedBlockingQueue<InboundMessage> captured;
    private EmailInboundConnector connector;

    @BeforeEach
    void setUp() {
        captured = new LinkedBlockingQueue<>();
        connector = new EmailInboundConnector(() -> List.of(testAccount()));
    }

    @AfterEach
    void tearDown() {
        connector.stop();
    }

    private EmailInboundAccount testAccount() {
        return new EmailInboundAccount(
                "email-inbound", "localhost", GREEN_MAIL.getImap().getPort(),
                false, "inbox@example.com", "password", "INBOX", 60);
    }

    /** Blocks until the IDLE loop delivers a message, fails after 3 s. */
    private InboundMessage receive() throws InterruptedException {
        final InboundMessage msg = captured.poll(3, TimeUnit.SECONDS);
        assertThat(msg).as("message not delivered within 3s — IDLE did not fire").isNotNull();
        return msg;
    }

    // ── helpers ──────────────────────────────────────────────────────────────

    private void deliver(final String from, final String subject, final String body) throws Exception {
        final MimeMessage msg = new MimeMessage(Session.getInstance(new Properties()));
        msg.setFrom(new InternetAddress(from));
        msg.setRecipient(Message.RecipientType.TO, new InternetAddress("inbox@example.com"));
        msg.setSubject(subject);
        msg.setText(body);
        msg.setSentDate(Date.from(Instant.now()));
        msg.setHeader("Message-ID", "<test-" + System.nanoTime() + "@example.com>");
        deliverViaSMTP(msg);
    }

    private void deliverViaSMTP(final MimeMessage msg) throws Exception {
        final Properties props = new Properties();
        props.put("mail.smtp.host", "localhost");
        props.put("mail.smtp.port", String.valueOf(GREEN_MAIL.getSmtp().getPort()));
        props.put("mail.smtp.auth", "true");
        try (final Transport transport = Session.getInstance(props).getTransport("smtp")) {
            transport.connect("inbox@example.com", "password");
            transport.sendMessage(msg, new jakarta.mail.Address[]{
                    new InternetAddress("inbox@example.com")});
        }
    }

    // Appends directly to IMAP mailbox — use for header-edge-case tests.
    // Pre-deliver BEFORE start() so processUnseen() on first connect catches it.
    private void deliverDirect(final MimeMessage msg) throws Exception {
        final GreenMailUser user = GREEN_MAIL.getUserManager().getUser("inbox@example.com");
        final MailFolder inbox = GREEN_MAIL.getManagers().getImapHostManager().getInbox(user);
        inbox.appendMessage(msg, new jakarta.mail.Flags(), new Date());
    }

    // ── identity and guard ────────────────────────────────────────────────────

    @Test
    void id_returnsEmailInbound() {
        assertThat(connector.id()).isEqualTo("email-inbound");
    }

    @Test
    @Timeout(5)
    void noAccounts_startIsNoOp_stopIsNoOp() {
        final EmailInboundConnector empty = new EmailInboundConnector(List::of);
        empty.start(captured::add);
        empty.stop();
        assertThat(captured).isEmpty();
    }

    @Test
    @Timeout(5)
    void doubleStart_isNoOp() throws Exception {
        connector.start(captured::add);
        connector.start(captured::add); // second call must not subscribe a second IDLE loop

        deliver("sender@example.com", "Subject", "Body");

        receive(); // exactly one delivery
        assertThat(captured.poll(500, TimeUnit.MILLISECONDS))
                .as("second delivery arrived — double-start guard failed")
                .isNull();
    }

    // ── delivery ─────────────────────────────────────────────────────────────

    @Test
    @Timeout(5)
    void singlePlainTextMessage_deliveredWithCorrectFields() throws Exception {
        connector.start(captured::add);
        deliver("sender@example.com", "Hello subject", "Hello body");

        final InboundMessage msg = receive();
        assertThat(msg.connectorId()).isEqualTo("email-inbound");
        assertThat(msg.externalSenderId()).isEqualTo("sender@example.com");
        assertThat(msg.externalChannelRef()).isEqualTo("inbox@example.com");
        assertThat(msg.content()).isEqualTo("Hello body");
        assertThat(msg.attachments()).isEmpty();
        assertThat(msg.receivedAt()).isNotNull();
        assertThat(msg.metadata()).containsEntry("account-id", "email-inbound");
        assertThat(msg.metadata()).containsEntry("subject", "Hello subject");
        assertThat(msg.metadata()).containsKey("message-id");
        assertThat(msg.metadata()).containsEntry("attachment-count", "0");
    }

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

    @Test
    @Timeout(10)
    void messageMarkedSeen_notRedeliveredAfterRestart() throws Exception {
        connector.start(captured::add);
        deliver("sender@example.com", "Once", "Only once");
        receive();
        connector.stop();

        final LinkedBlockingQueue<InboundMessage> second = new LinkedBlockingQueue<>();
        final EmailInboundConnector connector2 =
                new EmailInboundConnector(() -> List.of(testAccount()));
        try {
            connector2.start(second::add);
            assertThat(second.poll(1, TimeUnit.SECONDS)).as("message redelivered").isNull();
        } finally {
            connector2.stop();
        }
    }

    @Test
    @Timeout(5)
    void sinkThrows_messageStillMarkedSeen_remainingDelivered() throws Exception {
        deliver("a@example.com", "First", "Body A");
        deliver("b@example.com", "Second", "Body B");

        final LinkedBlockingQueue<String> contents = new LinkedBlockingQueue<>();
        final boolean[] first = {true};
        connector.start(msg -> {
            contents.add(msg.content());
            if (first[0]) { first[0] = false; throw new RuntimeException("Sink error"); }
        });

        assertThat(contents.poll(3, TimeUnit.SECONDS)).as("first message").isNotNull();
        assertThat(contents.poll(3, TimeUnit.SECONDS)).as("second message").isNotNull();

        // Neither redelivered after restart
        connector.stop();
        final LinkedBlockingQueue<InboundMessage> after = new LinkedBlockingQueue<>();
        final EmailInboundConnector connector2 =
                new EmailInboundConnector(() -> List.of(testAccount()));
        try {
            connector2.start(after::add);
            assertThat(after.poll(1, TimeUnit.SECONDS)).as("redelivery after sink-threw").isNull();
        } finally {
            connector2.stop();
        }
    }

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

    @Test
    @Timeout(5)
    void imapConnectionFailure_loggedAndNoSinkCall() throws Exception {
        final LinkedBlockingQueue<InboundMessage> q = new LinkedBlockingQueue<>();
        final EmailInboundConnector bad = new EmailInboundConnector(() -> List.of(
                new EmailInboundAccount("bad", "localhost", 19999, false,
                        "u", "p", "INBOX", 1)));
        bad.start(q::add);
        assertThat(q.poll(500, TimeUnit.MILLISECONDS)).isNull();
        bad.stop();
    }

    // ── edge cases (pre-deliver before start so processUnseen() on connect handles them) ──

    @Test
    @Timeout(5)
    void missingFromHeader_senderIdIsEmptyString() throws Exception {
        final MimeMessage msg = new MimeMessage(Session.getInstance(new Properties()));
        msg.setRecipient(Message.RecipientType.TO, new InternetAddress("inbox@example.com"));
        msg.setSubject("No from");
        msg.setText("Body");
        msg.setSentDate(Date.from(Instant.now()));
        deliverDirect(msg); // pre-deliver before start()

        connector.start(captured::add);
        assertThat(receive().externalSenderId()).isEmpty();
    }

    @Test
    @Timeout(5)
    void missingToHeader_channelRefFallsBackToAccountUsername() throws Exception {
        final MimeMessage msg = new MimeMessage(Session.getInstance(new Properties()));
        msg.setFrom(new InternetAddress("sender@example.com"));
        msg.setSubject("No To");
        msg.setText("BCC body");
        msg.setSentDate(Date.from(Instant.now()));
        deliverDirect(msg); // pre-deliver before start()

        connector.start(captured::add);
        assertThat(receive().externalChannelRef()).isEqualTo("inbox@example.com");
    }

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

    // ── buildMetadata (direct call — no IMAP needed) ──────────────────────────

    @Test
    void buildMetadata_noMessageIdHeader_keyAbsent() throws Exception {
        final MimeMessage msg = new MimeMessage(Session.getInstance(new Properties()));
        msg.setFrom(new InternetAddress("sender@example.com"));
        msg.setSubject("Has subject, no message-id");
        msg.setText("Body");

        final Map<String, String> metadata = EmailInboundConnector.buildMetadata(testAccount(), msg, 0);

        assertThat(metadata).containsKey("account-id");
        assertThat(metadata).doesNotContainKey("message-id");
        assertThat(metadata).containsEntry("subject", "Has subject, no message-id");
        assertThat(metadata).containsEntry("attachment-count", "0");
    }

    @Test
    void buildMetadata_noSubject_keyAbsent() throws Exception {
        final MimeMessage msg = new MimeMessage(Session.getInstance(new Properties()));
        msg.setFrom(new InternetAddress("sender@example.com"));
        msg.setHeader("Message-ID", "<test@example.com>");
        msg.setText("Body");

        final Map<String, String> metadata = EmailInboundConnector.buildMetadata(testAccount(), msg, 3);

        assertThat(metadata).containsEntry("account-id", EmailInboundConnector.ID);
        assertThat(metadata).containsEntry("message-id", "<test@example.com>");
        assertThat(metadata).doesNotContainKey("subject");
        assertThat(metadata).containsEntry("attachment-count", "3");
    }
}
```

- [ ] **Step 2: Run the new tests — expect many failures (no IDLE implementation yet)**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl email-inbound -Dtest=EmailInboundConnectorTest 2>&1 | tail -20
```

Expected: compilation errors (`executor`, `idleLoop`, `buildMetadata(account, msg, 0)` not found) or test failures. This is correct — TDD red phase.

- [ ] **Step 3: Replace `EmailInboundConnector.java` with IDLE implementation**

```java
package io.casehub.connectors.email.inbound;

import java.time.Instant;
import java.util.ArrayList;
import java.util.Collections;
import java.util.Date;
import java.util.LinkedHashMap;
import java.util.List;
import java.util.Map;
import java.util.Properties;
import java.util.concurrent.CopyOnWriteArrayList;
import java.util.concurrent.ExecutorService;
import java.util.concurrent.Executors;
import java.util.logging.Level;
import java.util.logging.Logger;

import jakarta.enterprise.context.ApplicationScoped;
import jakarta.inject.Inject;
import jakarta.mail.Address;
import jakarta.mail.Flags;
import jakarta.mail.Folder;
import jakarta.mail.FolderClosedException;
import jakarta.mail.Message;
import jakarta.mail.Session;
import jakarta.mail.Store;
import jakarta.mail.StoreClosedException;
import jakarta.mail.internet.InternetAddress;
import jakarta.mail.search.FlagTerm;

import org.eclipse.angus.mail.imap.IMAPFolder;

import io.casehub.connectors.InboundConnector;
import io.casehub.connectors.InboundMessage;
import io.casehub.connectors.InboundMessageSink;

/**
 * Pull-based inbound connector for IMAP mailboxes using IMAP IDLE (RFC 2177).
 *
 * <p>One virtual thread per account keeps a persistent IMAP connection and waits
 * for server push notifications. When the server notifies, all UNSEEN messages are
 * delivered to the sink and marked SEEN.
 *
 * <p>{@code connectorId} is always {@value #ID}. Per-account identity is in
 * {@code InboundMessage.metadata["account-id"]}.
 *
 * <h2>Delivery guarantee</h2>
 * At-least-once. If shutdown interrupts between {@code sink.receive()} and the
 * SEEN flag write, the message redelivers on next startup. Observers must be idempotent.
 *
 * <h2>Reconnection</h2>
 * Exponential backoff capped at {@code reconnectDelaySeconds}. Escalates to SEVERE
 * after 5+ consecutive failures. {@code FolderClosedException}/{@code StoreClosedException}
 * reconnect immediately at INFO — covers normal server-side IDLE timeouts.
 */
@ApplicationScoped
public class EmailInboundConnector implements InboundConnector {

    static final String ID = "email-inbound";

    private static final Logger LOG = Logger.getLogger(EmailInboundConnector.class.getName());

    private final EmailInboundAccountProvider provider;
    private final List<Store> openStores = new CopyOnWriteArrayList<>();
    private volatile boolean stopping = false;
    private ExecutorService executor;

    @Inject
    public EmailInboundConnector(final EmailInboundAccountProvider provider) {
        this.provider = provider;
    }

    @Override
    public String id() {
        return ID;
    }

    @Override
    public void start(final InboundMessageSink sink) {
        if (executor != null) return; // double-start guard
        executor = Executors.newVirtualThreadPerTaskExecutor();
        for (final EmailInboundAccount account : provider.accounts()) {
            executor.submit(() -> idleLoop(account, sink));
        }
    }

    @Override
    public void stop() {
        stopping = true;
        new ArrayList<>(openStores).forEach(store -> {
            try { store.close(); } catch (final Exception ignored) {}
        });
        if (executor != null) {
            executor.shutdownNow();
        }
    }

    private void idleLoop(final EmailInboundAccount account, final InboundMessageSink sink) {
        int backoffSeconds = 1;
        int consecutiveFailures = 0;

        while (!stopping) {
            Store store = null;
            IMAPFolder folder = null;
            try {
                store = connect(account);
                openStores.add(store);
                if (stopping) {
                    // Race guard: stop() may have snapshotted openStores before we added.
                    // Do not remove here — finally handles openStores.remove(store) and closeQuietly.
                    return;
                }
                folder = (IMAPFolder) store.getFolder(account.folder());
                folder.open(Folder.READ_WRITE);
                backoffSeconds = 1;
                consecutiveFailures = 0;
                LOG.info("email-inbound: IDLE connected for account " + account.id());

                // Catch messages already in the mailbox before IDLE started
                processUnseen(folder, account, sink);

                while (!stopping) {
                    folder.idle(false); // blocks until server notification or timeout
                    processUnseen(folder, account, sink);
                }

            } catch (final FolderClosedException | StoreClosedException e) {
                // Normal server-side IDLE timeout or server-closed connection.
                // consecutive quick disconnects will eventually fail to reconnect and hit the backoff path
                if (!stopping) {
                    LOG.info("email-inbound: IDLE session ended for account "
                            + account.id() + ", reconnecting");
                }
            } catch (final Exception e) {
                if (!stopping) {
                    consecutiveFailures++;
                    final Level level = consecutiveFailures >= 5 ? Level.SEVERE : Level.WARNING;
                    LOG.log(level, "email-inbound: connection failed for account "
                            + account.id() + " (attempt " + consecutiveFailures + "): "
                            + e.getMessage());
                    sleepQuietly(backoffSeconds * 1000L);
                    backoffSeconds = Math.min(backoffSeconds * 2, account.reconnectDelaySeconds());
                }
            } finally {
                openStores.remove(store);
                closeQuietly(folder, store);
            }
        }
    }

    private Store connect(final EmailInboundAccount account) throws Exception {
        final Properties props = buildProperties(account);
        final Session session = Session.getInstance(props);
        final Store store = session.getStore();
        store.connect(account.host(), account.username(), account.password());
        return store;
    }

    private void processUnseen(final IMAPFolder folder, final EmailInboundAccount account,
                                final InboundMessageSink sink) {
        try {
            final Message[] unseen = folder.search(
                    new FlagTerm(new Flags(Flags.Flag.SEEN), false));
            for (final Message msg : unseen) {
                final InboundMessage inbound = toInboundMessage(account, msg);
                try {
                    sink.receive(inbound);
                } catch (final Exception e) {
                    LOG.log(Level.SEVERE, "email-inbound: sink threw for account "
                            + account.id(), e);
                } finally {
                    try {
                        msg.setFlag(Flags.Flag.SEEN, true);
                    } catch (final Exception e) {
                        LOG.log(Level.WARNING, "email-inbound: failed to mark SEEN for account "
                                + account.id(), e);
                    }
                }
            }
        } catch (final Exception e) {
            LOG.log(Level.WARNING, "email-inbound: processUnseen failed for account "
                    + account.id(), e);
        }
    }

    private static Properties buildProperties(final EmailInboundAccount account) {
        final Properties props = new Properties();
        if (account.tls()) {
            props.put("mail.store.protocol", "imaps");
            props.put("mail.imaps.host", account.host());
            props.put("mail.imaps.port", String.valueOf(account.port()));
            props.put("mail.imaps.ssl.enable", "true");
            props.put("mail.imaps.timeout", "300000");
            props.put("mail.imaps.connectiontimeout", "30000");
        } else {
            props.put("mail.store.protocol", "imap");
            props.put("mail.imap.host", account.host());
            props.put("mail.imap.port", String.valueOf(account.port()));
            props.put("mail.imap.timeout", "300000");
            props.put("mail.imap.connectiontimeout", "30000");
        }
        return props;
    }

    private static InboundMessage toInboundMessage(final EmailInboundAccount account,
                                                    final Message msg) {
        try {
            return new InboundMessage(
                    ID,
                    extractSenderId(msg),
                    extractChannelRef(msg, account),
                    ContentExtractor.extractContent(msg),
                    resolveReceivedAt(msg),
                    buildMetadata(account, msg, 0));
        } catch (final Exception e) {
            LOG.log(Level.WARNING, "email-inbound: message parse failed", e);
            return new InboundMessage(ID, "", account.username(), "",
                    Instant.now(), Map.of("account-id", account.id()));
        }
    }

    private static String extractSenderId(final Message msg) {
        try {
            final Address[] from = msg.getFrom();
            if (from != null && from.length > 0 && from[0] instanceof final InternetAddress ia) {
                final String addr = ia.getAddress();
                if (addr != null) return addr;
            }
        } catch (final Exception ignored) {}
        return "";
    }

    private static String extractChannelRef(final Message msg,
                                            final EmailInboundAccount account) {
        try {
            final Address[] to = msg.getRecipients(Message.RecipientType.TO);
            if (to != null && to.length > 0 && to[0] instanceof final InternetAddress ia) {
                final String addr = ia.getAddress();
                if (addr != null && !addr.isBlank()) return addr;
            }
        } catch (final Exception ignored) {}
        return account.username();
    }

    private static Instant resolveReceivedAt(final Message msg) {
        try {
            final Date received = msg.getReceivedDate();
            if (received != null) return received.toInstant();
            final Date sent = msg.getSentDate();
            if (sent != null) return sent.toInstant();
        } catch (final Exception ignored) {}
        return Instant.now();
    }

    static Map<String, String> buildMetadata(final EmailInboundAccount account,
                                              final Message msg,
                                              final int attachmentCount) {
        final Map<String, String> meta = new LinkedHashMap<>();
        meta.put("account-id", account.id());
        try {
            final String[] msgId = msg.getHeader("Message-ID");
            if (msgId != null && msgId.length > 0 && msgId[0] != null) {
                meta.put("message-id", msgId[0].trim());
            }
            final String subject = msg.getSubject();
            if (subject != null) {
                meta.put("subject", subject);
            }
        } catch (final Exception ignored) {}
        meta.put("attachment-count", String.valueOf(attachmentCount));
        return Collections.unmodifiableMap(meta);
    }

    private static void sleepQuietly(final long millis) {
        try {
            Thread.sleep(millis);
        } catch (final InterruptedException e) {
            Thread.currentThread().interrupt();
        }
    }

    private static void closeQuietly(final Folder folder, final Store store) {
        if (folder != null) {
            try { folder.close(false); } catch (final Exception ignored) {}
        }
        if (store != null) {
            try { store.close(); } catch (final Exception ignored) {}
        }
    }
}
```

- [ ] **Step 4: Run `EmailInboundConnectorTest` — expect green**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl email-inbound -Dtest=EmailInboundConnectorTest
```

Expected: BUILD SUCCESS, all tests pass

- [ ] **Step 5: Update `GreenMailResource.java` — drop poll-interval config key**

```java
package io.casehub.connectors.email.inbound;

import java.util.Map;

import com.icegreen.greenmail.configuration.GreenMailConfiguration;
import com.icegreen.greenmail.util.GreenMail;
import com.icegreen.greenmail.util.ServerSetupTest;
import io.quarkus.test.common.QuarkusTestResourceLifecycleManager;

public class GreenMailResource implements QuarkusTestResourceLifecycleManager {

    static GreenMail INSTANCE;

    @Override
    public Map<String, String> start() {
        INSTANCE = new GreenMail(ServerSetupTest.SMTP_IMAP);
        INSTANCE.withConfiguration(GreenMailConfiguration.aConfig()
                .withUser("inbox@example.com", "password"));
        INSTANCE.start();
        return Map.of(
                "casehub.connectors.email-inbound.host", "localhost",
                "casehub.connectors.email-inbound.port",
                        String.valueOf(INSTANCE.getImap().getPort()),
                "casehub.connectors.email-inbound.tls", "false",
                "casehub.connectors.email-inbound.username", "inbox@example.com",
                "casehub.connectors.email-inbound.password", "password");
        // reconnect-delay-seconds defaults to 60 — no need to set it here
    }

    @Override
    public void stop() {
        if (INSTANCE != null) INSTANCE.stop();
    }
}
```

- [ ] **Step 6: Run the Quarkus integration test**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl email-inbound -Dtest=EmailInboundConnectorQuarkusTest
```

Expected: BUILD SUCCESS. The test sends via SMTP and waits 2 s; IDLE delivers it well within that window.

- [ ] **Step 7: Run the full email-inbound module**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl email-inbound
```

Expected: BUILD SUCCESS, all tests pass

- [ ] **Step 8: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/connectors add \
  email-inbound/src/main/java/io/casehub/connectors/email/inbound/EmailInboundConnector.java \
  email-inbound/src/test/java/io/casehub/connectors/email/inbound/EmailInboundConnectorTest.java \
  email-inbound/src/test/java/io/casehub/connectors/email/inbound/GreenMailResource.java
git -C /Users/mdproctor/claude/casehub/connectors commit -m "feat(email-inbound): replace IMAP polling with IDLE via VirtualThreadPerTaskExecutor — Closes #9"
```

---

## Phase 2 — Issue #10: Attachment Support

---

### Task 3: Add `Attachment` record to `core`

**Files:**
- Create: `core/src/main/java/io/casehub/connectors/Attachment.java`
- Modify: `core/src/test/java/io/casehub/connectors/` (add `AttachmentTest.java` if the directory exists; otherwise add it adjacent to `ConnectorTest.java`)

The record overrides `equals()`, `hashCode()`, and `toString()` because Java records use `Objects.equals()` for field comparison, which for `byte[]` is reference equality — not content equality. AssertJ's `containsExactly` would fail even with identical bytes without these overrides.

- [ ] **Step 1: Write the failing test — create `AttachmentTest.java`**

Path: `core/src/test/java/io/casehub/connectors/AttachmentTest.java`

```java
package io.casehub.connectors;

import static org.assertj.core.api.Assertions.assertThat;

import org.junit.jupiter.api.Test;

class AttachmentTest {

    @Test
    void construction_storesAllFields() {
        final byte[] bytes = {1, 2, 3};
        final Attachment att = new Attachment("report.pdf", "application/pdf", bytes);

        assertThat(att.filename()).isEqualTo("report.pdf");
        assertThat(att.contentType()).isEqualTo("application/pdf");
        assertThat(att.content()).isEqualTo(new byte[]{1, 2, 3});
    }

    @Test
    void nullFilename_isAllowed() {
        final Attachment att = new Attachment(null, "application/octet-stream", new byte[]{});
        assertThat(att.filename()).isNull();
    }

    @Test
    void nullContent_treatedAsEmptyArray() {
        final Attachment att = new Attachment("f.pdf", "application/pdf", null);
        assertThat(att.content()).isEqualTo(new byte[0]);
    }

    @Test
    void content_defensivelyCopiedOnConstruction() {
        final byte[] original = {10, 20, 30};
        final Attachment att = new Attachment("f.bin", "application/octet-stream", original);
        original[0] = 99;
        assertThat(att.content()[0]).isEqualTo((byte) 10); // stored copy unaffected
    }

    @Test
    void content_defensivelyCopiedOnAccess() {
        final Attachment att = new Attachment("f.bin", "application/octet-stream", new byte[]{1, 2});
        final byte[] returned = att.content();
        returned[0] = 99;
        assertThat(att.content()[0]).isEqualTo((byte) 1); // stored copy unaffected
    }

    @Test
    void equals_sameFields_areEqual() {
        final Attachment a = new Attachment("f.pdf", "application/pdf", new byte[]{1, 2});
        final Attachment b = new Attachment("f.pdf", "application/pdf", new byte[]{1, 2});
        assertThat(a).isEqualTo(b);
    }

    @Test
    void equals_differentContent_notEqual() {
        final Attachment a = new Attachment("f.pdf", "application/pdf", new byte[]{1, 2});
        final Attachment b = new Attachment("f.pdf", "application/pdf", new byte[]{3, 4});
        assertThat(a).isNotEqualTo(b);
    }

    @Test
    void hashCode_equalAttachments_samHashCode() {
        final Attachment a = new Attachment("f.pdf", "application/pdf", new byte[]{1, 2});
        final Attachment b = new Attachment("f.pdf", "application/pdf", new byte[]{1, 2});
        assertThat(a.hashCode()).isEqualTo(b.hashCode());
    }

    @Test
    void toString_includesByteCount_notRawArray() {
        final Attachment att = new Attachment("report.pdf", "application/pdf", new byte[]{1, 2, 3});
        assertThat(att.toString()).contains("report.pdf").contains("application/pdf").contains("3");
        assertThat(att.toString()).doesNotContain("[B@"); // no default Object.toString() on byte[]
    }
}
```

- [ ] **Step 2: Run the test — expect failure (class not found)**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl core -Dtest=AttachmentTest 2>&1 | tail -5
```

Expected: compilation error — `Attachment` does not exist yet.

- [ ] **Step 3: Create `Attachment.java`**

Path: `core/src/main/java/io/casehub/connectors/Attachment.java`

```java
package io.casehub.connectors;

import java.util.Arrays;
import java.util.Objects;

/**
 * An attachment extracted from an inbound message.
 *
 * <p>{@code filename} is nullable — MIME does not require a filename.
 * {@code contentType} is the base MIME type, parameters stripped and lowercased
 * (e.g. {@code "application/pdf"}, never {@code "Application/PDF; name=invoice.pdf"}).
 * {@code content} is defensively copied on construction and access.
 *
 * <h2>V1 constraint</h2>
 * Content is fully materialised into a heap-resident {@code byte[]}. Callers
 * processing large files (multi-MB PDFs, scans) must account for heap pressure.
 * Streaming is deferred to a future version.
 */
public record Attachment(String filename, String contentType, byte[] content) {

    public Attachment {
        content = content == null ? new byte[0] : content.clone();
    }

    @Override
    public byte[] content() {
        return content.clone();
    }

    @Override
    public boolean equals(final Object obj) {
        if (this == obj) return true;
        if (!(obj instanceof final Attachment other)) return false;
        return Objects.equals(filename, other.filename)
                && Objects.equals(contentType, other.contentType)
                && Arrays.equals(content, other.content);
    }

    @Override
    public int hashCode() {
        return Objects.hash(filename, contentType, Arrays.hashCode(content));
    }

    @Override
    public String toString() {
        return "Attachment[filename=" + filename
                + ", contentType=" + contentType
                + ", content=" + content.length + " bytes]";
    }
}
```

- [ ] **Step 4: Run `AttachmentTest` — expect green**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl core -Dtest=AttachmentTest
```

Expected: BUILD SUCCESS, all 9 tests pass

- [ ] **Step 5: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/connectors add \
  core/src/main/java/io/casehub/connectors/Attachment.java \
  core/src/test/java/io/casehub/connectors/AttachmentTest.java
git -C /Users/mdproctor/claude/casehub/connectors commit -m "feat(core): add Attachment record with defensive byte[] copy — Refs #10"
```

---

### Task 4: Extend `InboundMessage` with `attachments` field

**Files:**
- Modify: `core/src/main/java/io/casehub/connectors/InboundMessage.java`

The canonical 7-arg constructor wraps `attachments` with `List.copyOf()` — required because `ExtractionResult` returns a mutable `ArrayList`. The two convenience constructors preserve all existing webhook connector call sites (`SlackInboundConnector` etc.) without any changes to those files.

- [ ] **Step 1: Update the test — open `ConnectorTest.java` or `InboundConnectorServiceTest.java` to find the canonical test location, then add an `InboundMessageTest` if one doesn't exist**

Create `core/src/test/java/io/casehub/connectors/InboundMessageTest.java`:

```java
package io.casehub.connectors;

import static org.assertj.core.api.Assertions.assertThat;

import java.time.Instant;
import java.util.ArrayList;
import java.util.List;
import java.util.Map;

import org.junit.jupiter.api.Test;

class InboundMessageTest {

    @Test
    void sevenArgConstructor_allFieldsSet() {
        final List<Attachment> atts = List.of(
                new Attachment("f.pdf", "application/pdf", new byte[]{1}));
        final Instant now = Instant.now();
        final InboundMessage msg = new InboundMessage(
                "email-inbound", "sender@example.com", "inbox@example.com",
                "body", atts, now, Map.of("k", "v"));

        assertThat(msg.connectorId()).isEqualTo("email-inbound");
        assertThat(msg.externalSenderId()).isEqualTo("sender@example.com");
        assertThat(msg.externalChannelRef()).isEqualTo("inbox@example.com");
        assertThat(msg.content()).isEqualTo("body");
        assertThat(msg.attachments()).hasSize(1);
        assertThat(msg.receivedAt()).isEqualTo(now);
        assertThat(msg.metadata()).containsEntry("k", "v");
    }

    @Test
    void sixArgConvenienceConstructor_attachmentsEmpty() {
        final InboundMessage msg = new InboundMessage(
                "slack-inbound", "U123", "C456", "hello", Instant.now(), Map.of());
        assertThat(msg.attachments()).isEmpty();
    }

    @Test
    void fiveArgConvenienceConstructor_attachmentsEmptyMetadataEmpty() {
        final InboundMessage msg = new InboundMessage(
                "slack-inbound", "U123", "C456", "hello", Instant.now());
        assertThat(msg.attachments()).isEmpty();
        assertThat(msg.metadata()).isEmpty();
    }

    @Test
    void attachments_defensivelyCopied_mutableListCannotAffectRecord() {
        final List<Attachment> mutable = new ArrayList<>();
        mutable.add(new Attachment("f.pdf", "application/pdf", new byte[]{1}));
        final InboundMessage msg = new InboundMessage(
                "email-inbound", "s", "c", "body", mutable, Instant.now(), Map.of());

        mutable.clear(); // mutate after construction
        assertThat(msg.attachments()).hasSize(1); // record unaffected
    }
}
```

- [ ] **Step 2: Run — expect failure (no `attachments` field yet)**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl core -Dtest=InboundMessageTest 2>&1 | tail -5
```

Expected: compilation error.

- [ ] **Step 3: Replace `InboundMessage.java`**

```java
package io.casehub.connectors;

import java.time.Instant;
import java.util.List;
import java.util.Map;

/**
 * A message received from an external system via an {@link InboundConnector}.
 *
 * <p>{@code attachments} is always non-null; it is {@code List.of()} for connectors
 * that produce no attachments (Slack, Teams, SMS, WhatsApp). Email inbound
 * populates it from the MIME structure.
 *
 * <p>{@code metadata["attachment-count"]} is always present for email-inbound messages
 * (even when zero), allowing observers to branch without touching binary content.
 */
public record InboundMessage(
        String connectorId,
        String externalSenderId,
        String externalChannelRef,
        String content,
        List<Attachment> attachments,
        Instant receivedAt,
        Map<String, String> metadata) {

    public InboundMessage {
        attachments = List.copyOf(attachments);
    }

    /** No attachments, with metadata. Preserves all existing webhook connector call sites. */
    public InboundMessage(final String connectorId,
                          final String externalSenderId,
                          final String externalChannelRef,
                          final String content,
                          final Instant receivedAt,
                          final Map<String, String> metadata) {
        this(connectorId, externalSenderId, externalChannelRef, content,
                List.of(), receivedAt, metadata);
    }

    /** No attachments, no metadata. */
    public InboundMessage(final String connectorId,
                          final String externalSenderId,
                          final String externalChannelRef,
                          final String content,
                          final Instant receivedAt) {
        this(connectorId, externalSenderId, externalChannelRef, content,
                List.of(), receivedAt, Map.of());
    }
}
```

- [ ] **Step 4: Run `InboundMessageTest` — expect green**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl core -Dtest=InboundMessageTest
```

Expected: BUILD SUCCESS

- [ ] **Step 5: Run the full build to verify all modules still compile**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn clean install -q
```

Expected: BUILD SUCCESS — webhook connectors use the 6-arg convenience constructor and compile unchanged.

- [ ] **Step 6: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/connectors add \
  core/src/main/java/io/casehub/connectors/InboundMessage.java \
  core/src/test/java/io/casehub/connectors/InboundMessageTest.java
git -C /Users/mdproctor/claude/casehub/connectors commit -m "feat(core): add List<Attachment> attachments field to InboundMessage — Refs #10"
```

---

### Task 5: Create `ExtractionResult` and refactor `ContentExtractor` to single-pass

**Files:**
- Create: `email-inbound/src/main/java/io/casehub/connectors/email/inbound/ExtractionResult.java`
- Modify: `email-inbound/src/main/java/io/casehub/connectors/email/inbound/ContentExtractor.java`
- Modify: `email-inbound/src/test/java/io/casehub/connectors/email/inbound/ContentExtractorTest.java`

`ContentExtractor.extractContent()` is removed and replaced by `ContentExtractor.extract()` returning `ExtractionResult`. The existing four `ContentExtractorTest` cases are updated to call `extract().content()` and new cases are added for attachment extraction. The old `extractContent()` method must not remain (it will cause a compilation error in Task 6 when `toInboundMessage()` is updated).

- [ ] **Step 1: Replace `ContentExtractorTest.java` with the updated + extended test suite**

```java
package io.casehub.connectors.email.inbound;

import static org.assertj.core.api.Assertions.assertThat;

import java.util.Properties;

import jakarta.mail.Part;
import jakarta.mail.Session;
import jakarta.mail.internet.MimeBodyPart;
import jakarta.mail.internet.MimeMessage;
import jakarta.mail.internet.MimeMultipart;

import io.casehub.connectors.Attachment;

import org.junit.jupiter.api.Test;

// Note: newly-constructed MimeMessage objects default Content-Type to "text/plain" until
// saveChanges() is called. Messages fetched from IMAP already have correct headers — so
// the implementation is correct. Tests call saveChanges() to simulate that committed state.

class ContentExtractorTest {

    private static Session emptySession() {
        return Session.getInstance(new Properties());
    }

    // ── text content (existing cases updated to ExtractionResult API) ──────────

    @Test
    void plainText_returnedInContent_noAttachments() throws Exception {
        final MimeMessage msg = new MimeMessage(emptySession());
        msg.setText("Hello world", "UTF-8");

        final ExtractionResult result = ContentExtractor.extract(msg);
        assertThat(result.content()).isEqualTo("Hello world");
        assertThat(result.attachments()).isEmpty();
    }

    @Test
    void htmlOnly_returnedAsRawHtml_noAttachments() throws Exception {
        final MimeMessage msg = new MimeMessage(emptySession());
        msg.setContent("<p>Hello</p>", "text/html; charset=UTF-8");

        final ExtractionResult result = ContentExtractor.extract(msg);
        assertThat(result.content()).isEqualTo("<p>Hello</p>");
        assertThat(result.attachments()).isEmpty();
    }

    @Test
    void multipartAlternative_prefersPlainText() throws Exception {
        final MimeMessage msg = new MimeMessage(emptySession());
        final MimeMultipart mp = new MimeMultipart("alternative");

        final MimeBodyPart plain = new MimeBodyPart();
        plain.setText("Plain text body", "UTF-8");

        final MimeBodyPart html = new MimeBodyPart();
        html.setContent("<p>HTML body</p>", "text/html; charset=UTF-8");

        mp.addBodyPart(plain);
        mp.addBodyPart(html);
        msg.setContent(mp);
        msg.saveChanges();

        final ExtractionResult result = ContentExtractor.extract(msg);
        assertThat(result.content()).isEqualTo("Plain text body");
        assertThat(result.attachments()).isEmpty();
    }

    @Test
    void multipartMixedWithNestedAlternative_extractsPlainTextAndIgnoresTextAttachmentInBody() throws Exception {
        final MimeMessage msg = new MimeMessage(emptySession());
        final MimeMultipart mixed = new MimeMultipart("mixed");

        final MimeMultipart alternative = new MimeMultipart("alternative");
        final MimeBodyPart plain = new MimeBodyPart();
        plain.setText("Plain body", "UTF-8");
        final MimeBodyPart html = new MimeBodyPart();
        html.setContent("<p>HTML body</p>", "text/html; charset=UTF-8");
        alternative.addBodyPart(plain);
        alternative.addBodyPart(html);
        final MimeBodyPart alternativeWrapper = new MimeBodyPart();
        alternativeWrapper.setContent(alternative);
        mixed.addBodyPart(alternativeWrapper);

        // Binary attachment alongside text body
        final MimeBodyPart attachment = new MimeBodyPart();
        attachment.setContent("pdf bytes".getBytes(), "application/pdf");
        attachment.setDisposition(Part.ATTACHMENT);
        attachment.setFileName("report.pdf");
        mixed.addBodyPart(attachment);

        msg.setContent(mixed);
        msg.saveChanges();

        final ExtractionResult result = ContentExtractor.extract(msg);
        assertThat(result.content()).isEqualTo("Plain body");
        assertThat(result.attachments()).hasSize(1);
        assertThat(result.attachments().get(0).filename()).isEqualTo("report.pdf");
        assertThat(result.attachments().get(0).contentType()).isEqualTo("application/pdf");
    }

    // ── attachment extraction ──────────────────────────────────────────────────

    @Test
    void multipartWithOnlyBinaryAttachment_emptyContent_attachmentPresent() throws Exception {
        final MimeMessage msg = new MimeMessage(emptySession());
        final MimeMultipart mixed = new MimeMultipart("mixed");

        final MimeBodyPart att = new MimeBodyPart();
        att.setContent(new byte[]{1, 2, 3}, "application/pdf");
        att.setDisposition(Part.ATTACHMENT);
        att.setFileName("file.pdf");
        mixed.addBodyPart(att);

        msg.setContent(mixed);
        msg.saveChanges();

        final ExtractionResult result = ContentExtractor.extract(msg);
        assertThat(result.content()).isEmpty();
        assertThat(result.attachments()).hasSize(1);
        assertThat(result.attachments().get(0).filename()).isEqualTo("file.pdf");
        assertThat(result.attachments().get(0).contentType()).isEqualTo("application/pdf");
        assertThat(result.attachments().get(0).content()).isEqualTo(new byte[]{1, 2, 3});
    }

    @Test
    void multipleAttachments_allCollected() throws Exception {
        final MimeMessage msg = new MimeMessage(emptySession());
        final MimeMultipart mixed = new MimeMultipart("mixed");

        final MimeBodyPart textPart = new MimeBodyPart();
        textPart.setText("Body", "UTF-8");
        mixed.addBodyPart(textPart);

        final MimeBodyPart pdf = new MimeBodyPart();
        pdf.setContent(new byte[]{1}, "application/pdf");
        pdf.setDisposition(Part.ATTACHMENT);
        pdf.setFileName("a.pdf");
        mixed.addBodyPart(pdf);

        final MimeBodyPart img = new MimeBodyPart();
        img.setContent(new byte[]{2, 3}, "image/png");
        img.setDisposition(Part.ATTACHMENT);
        img.setFileName("b.png");
        mixed.addBodyPart(img);

        msg.setContent(mixed);
        msg.saveChanges();

        final ExtractionResult result = ContentExtractor.extract(msg);
        assertThat(result.content()).isEqualTo("Body");
        assertThat(result.attachments()).hasSize(2);
        assertThat(result.attachments()).extracting(Attachment::filename)
                .containsExactlyInAnyOrder("a.pdf", "b.png");
    }

    @Test
    void attachmentWithNoFilename_filenameIsNull() throws Exception {
        final MimeMessage msg = new MimeMessage(emptySession());
        final MimeMultipart mixed = new MimeMultipart("mixed");

        final MimeBodyPart att = new MimeBodyPart();
        att.setContent(new byte[]{9}, "application/octet-stream");
        att.setDisposition(Part.ATTACHMENT);
        // no setFileName()
        mixed.addBodyPart(att);

        msg.setContent(mixed);
        msg.saveChanges();

        final ExtractionResult result = ContentExtractor.extract(msg);
        assertThat(result.attachments()).hasSize(1);
        assertThat(result.attachments().get(0).filename()).isNull();
    }

    @Test
    void contentTypeParametersStripped_lowercased() throws Exception {
        final MimeMessage msg = new MimeMessage(emptySession());
        final MimeMultipart mixed = new MimeMultipart("mixed");

        final MimeBodyPart att = new MimeBodyPart();
        // Content-Type with parameters and mixed case
        att.setContent(new byte[]{1}, "Application/PDF; name=\"invoice.pdf\"");
        att.setDisposition(Part.ATTACHMENT);
        att.setFileName("invoice.pdf");
        mixed.addBodyPart(att);

        msg.setContent(mixed);
        msg.saveChanges();

        final ExtractionResult result = ContentExtractor.extract(msg);
        assertThat(result.attachments().get(0).contentType()).isEqualTo("application/pdf");
    }

    @Test
    void textCalendar_collectedAsAttachment_notContent() throws Exception {
        // text/calendar (iCal) is a text/* subtype but not text/plain or text/html —
        // it goes to attachments so observers can process it as structured data
        final MimeMessage msg = new MimeMessage(emptySession());
        final MimeMultipart mixed = new MimeMultipart("mixed");

        final MimeBodyPart body = new MimeBodyPart();
        body.setText("See invite", "UTF-8");
        mixed.addBodyPart(body);

        final MimeBodyPart cal = new MimeBodyPart();
        cal.setContent("BEGIN:VCALENDAR\nEND:VCALENDAR", "text/calendar; method=REQUEST");
        cal.setDisposition(Part.ATTACHMENT);
        cal.setFileName("invite.ics");
        mixed.addBodyPart(cal);

        msg.setContent(mixed);
        msg.saveChanges();

        final ExtractionResult result = ContentExtractor.extract(msg);
        assertThat(result.content()).isEqualTo("See invite");
        assertThat(result.attachments()).hasSize(1);
        assertThat(result.attachments().get(0).contentType()).isEqualTo("text/calendar");
    }
}
```

- [ ] **Step 2: Run — expect failure (`ContentExtractor.extract()` and `ExtractionResult` don't exist)**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl email-inbound -Dtest=ContentExtractorTest 2>&1 | tail -10
```

Expected: compilation error.

- [ ] **Step 3: Create `ExtractionResult.java`**

```java
package io.casehub.connectors.email.inbound;

import java.util.List;

import io.casehub.connectors.Attachment;

record ExtractionResult(String content, List<Attachment> attachments) {}
```

- [ ] **Step 4: Replace `ContentExtractor.java` with single-pass implementation**

```java
package io.casehub.connectors.email.inbound;

import java.io.IOException;
import java.util.ArrayList;
import java.util.List;
import java.util.logging.Level;
import java.util.logging.Logger;

import jakarta.mail.MessagingException;
import jakarta.mail.Multipart;
import jakarta.mail.Part;

import io.casehub.connectors.Attachment;

/**
 * Single-pass recursive MIME content extractor.
 *
 * <p>Traverses the MIME tree once, collecting:
 * <ul>
 *   <li>{@code text/plain} → body content (preferred)</li>
 *   <li>{@code text/html} → body content fallback</li>
 *   <li>anything else → {@link Attachment} (filename, base content-type, bytes)</li>
 * </ul>
 *
 * <p>This includes {@code text/calendar}, {@code text/csv}, {@code text/x-vcard},
 * and inline binary parts — any non-plain/non-html MIME type goes to attachments.
 * Observers decide what to do with them; the connector delivers everything present.
 */
final class ContentExtractor {

    private static final Logger LOG = Logger.getLogger(ContentExtractor.class.getName());

    private ContentExtractor() {}

    static ExtractionResult extract(final Part part) {
        final Accumulator acc = new Accumulator();
        traverse(part, acc);
        return new ExtractionResult(acc.resolveContent(), List.copyOf(acc.attachments));
    }

    private static void traverse(final Part part, final Accumulator acc) {
        try {
            if (part.isMimeType("text/plain") && acc.plainText == null) {
                acc.plainText = part.getContent().toString();
            } else if (part.isMimeType("text/html") && acc.htmlText == null) {
                acc.htmlText = part.getContent().toString();
            } else if (part.isMimeType("multipart/*")) {
                final Multipart mp = (Multipart) part.getContent();
                for (int i = 0; i < mp.getCount(); i++) {
                    traverse(mp.getBodyPart(i), acc);
                }
            } else {
                acc.attachments.add(toAttachment(part));
            }
        } catch (final Exception e) {
            LOG.log(Level.WARNING, "email-inbound: content extraction failed on part", e);
        }
    }

    private static Attachment toAttachment(final Part part)
            throws MessagingException, IOException {
        final String filename = part.getFileName(); // null if absent
        String ct = part.getContentType();
        if (ct == null) ct = "application/octet-stream"; // RFC 2045 §5.2 default
        final String baseType = ct.contains(";")
                ? ct.substring(0, ct.indexOf(';')).trim().toLowerCase()
                : ct.trim().toLowerCase();
        final byte[] bytes = part.getInputStream().readAllBytes();
        return new Attachment(filename, baseType, bytes);
    }

    private static final class Accumulator {
        String plainText = null;
        String htmlText  = null;
        final List<Attachment> attachments = new ArrayList<>();

        String resolveContent() {
            if (plainText != null) return plainText;
            if (htmlText  != null) return htmlText;
            return "";
        }
    }
}
```

- [ ] **Step 5: Run `ContentExtractorTest` — expect green**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl email-inbound -Dtest=ContentExtractorTest
```

Expected: BUILD SUCCESS, all tests pass

- [ ] **Step 6: Verify the rest of `email-inbound` still compiles**

`EmailInboundConnector` still calls the now-removed `ContentExtractor.extractContent()` — it will fail to compile. This is expected at this stage; Task 6 fixes it.

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn compile -pl email-inbound 2>&1 | tail -10
```

Expected: compilation error on `ContentExtractor.extractContent` — this is correct. Do NOT fix it here; Task 6 does.

- [ ] **Step 7: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/connectors add \
  email-inbound/src/main/java/io/casehub/connectors/email/inbound/ExtractionResult.java \
  email-inbound/src/main/java/io/casehub/connectors/email/inbound/ContentExtractor.java \
  email-inbound/src/test/java/io/casehub/connectors/email/inbound/ContentExtractorTest.java
git -C /Users/mdproctor/claude/casehub/connectors commit -m "refactor(email-inbound): single-pass ContentExtractor returning ExtractionResult(content, attachments) — Refs #10"
```

---

### Task 6: Wire attachment extraction into `EmailInboundConnector` + update Quarkus test

**Files:**
- Modify: `email-inbound/src/main/java/io/casehub/connectors/email/inbound/EmailInboundConnector.java`
- Modify: `email-inbound/src/test/java/io/casehub/connectors/email/inbound/EmailInboundConnectorTest.java`
- Modify: `email-inbound/src/test/java/io/casehub/connectors/email/inbound/EmailInboundConnectorQuarkusTest.java`

- [ ] **Step 1: Add attachment tests to `EmailInboundConnectorTest.java`**

Add the following imports at the top of the class (add alongside existing imports):

```java
import jakarta.mail.internet.MimeBodyPart;
import jakarta.mail.internet.MimeMultipart;
import io.casehub.connectors.Attachment;
```

Add these test methods to the end of `EmailInboundConnectorTest`:

```java
    // ── attachment delivery (Phase 2) ─────────────────────────────────────────

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

    @Test
    @Timeout(5)
    void messageWithNoAttachments_attachmentsEmptyAndCountIsZero() throws Exception {
        connector.start(captured::add);
        deliver("sender@example.com", "Plain", "Body");

        final InboundMessage msg = receive();
        assertThat(msg.attachments()).isEmpty();
        assertThat(msg.metadata()).containsEntry("attachment-count", "0");
    }

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

- [ ] **Step 2: Run just the new attachment tests — expect failure (toInboundMessage still passes 0 for attachmentCount)**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl email-inbound \
  -Dtest="EmailInboundConnectorTest#messageWithPdfAttachment_attachmentDelivered" 2>&1 | tail -15
```

Expected: compilation error (because `ContentExtractor.extractContent()` no longer exists).

- [ ] **Step 3: Update `toInboundMessage()` in `EmailInboundConnector.java`**

Replace the `toInboundMessage()` method:

```java
    private static InboundMessage toInboundMessage(final EmailInboundAccount account,
                                                    final Message msg) {
        try {
            final ExtractionResult extracted = ContentExtractor.extract(msg);
            return new InboundMessage(
                    ID,
                    extractSenderId(msg),
                    extractChannelRef(msg, account),
                    extracted.content(),
                    extracted.attachments(),
                    resolveReceivedAt(msg),
                    buildMetadata(account, msg, extracted.attachments().size()));
        } catch (final Exception e) {
            LOG.log(Level.WARNING, "email-inbound: message parse failed", e);
            return new InboundMessage(ID, "", account.username(), "",
                    Instant.now(), Map.of("account-id", account.id()));
        }
    }
```

Also remove the old import `import jakarta.mail.Multipart;` and `import jakarta.mail.Part;` if they appear — `ContentExtractor` now owns all Part handling. Keep `import jakarta.mail.Message;` and `import jakarta.mail.search.FlagTerm;`.

- [ ] **Step 4: Run the full `EmailInboundConnectorTest` — expect all tests green**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl email-inbound -Dtest=EmailInboundConnectorTest
```

Expected: BUILD SUCCESS

- [ ] **Step 5: Update `EmailInboundConnectorQuarkusTest.java` — add attachment assertions**

Add two assertions to the existing `happyPath_emailDelivered_cdiFired` test, after the existing assertions:

```java
        assertThat(delivered.attachments()).isEmpty();
        assertThat(delivered.metadata()).containsEntry("attachment-count", "0");
```

- [ ] **Step 6: Run the Quarkus integration test**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl email-inbound -Dtest=EmailInboundConnectorQuarkusTest
```

Expected: BUILD SUCCESS

- [ ] **Step 7: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/connectors add \
  email-inbound/src/main/java/io/casehub/connectors/email/inbound/EmailInboundConnector.java \
  email-inbound/src/test/java/io/casehub/connectors/email/inbound/EmailInboundConnectorTest.java \
  email-inbound/src/test/java/io/casehub/connectors/email/inbound/EmailInboundConnectorQuarkusTest.java
git -C /Users/mdproctor/claude/casehub/connectors commit -m "feat(email-inbound): wire attachment extraction into EmailInboundConnector — Closes #10"
```

---

### Task 7: Full build verification

- [ ] **Step 1: Run the complete build**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn clean install
```

Expected: BUILD SUCCESS, all modules pass. Spot-check the test count — should be higher than before this branch started (new tests added in core + email-inbound).

- [ ] **Step 2: Verify issues are linked in commit history**

```bash
git -C /Users/mdproctor/claude/casehub/connectors log --oneline -6
```

Expected output (newest first):
```
<sha> feat(email-inbound): wire attachment extraction into EmailInboundConnector — Closes #10
<sha> refactor(email-inbound): single-pass ContentExtractor returning ExtractionResult — Refs #10
<sha> feat(core): add List<Attachment> attachments field to InboundMessage — Refs #10
<sha> feat(core): add Attachment record with defensive byte[] copy — Refs #10
<sha> feat(email-inbound): replace IMAP polling with IDLE via VirtualThreadPerTaskExecutor — Closes #9
<sha> refactor(email-inbound): rename pollIntervalSeconds → reconnectDelaySeconds — Refs #9
```
