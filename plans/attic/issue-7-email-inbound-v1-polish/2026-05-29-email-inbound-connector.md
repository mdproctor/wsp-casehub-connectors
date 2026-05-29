# Email Inbound Connector Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add a new `email-inbound` Maven module that polls IMAP mailboxes and delivers received emails as `InboundMessage` CDI events via `InboundConnectorService`.

**Architecture:** `EmailInboundConnector implements InboundConnector` discovers accounts from `EmailInboundAccountProvider` SPI; for each account, a single-threaded `ScheduledExecutorService` runs a poll cycle (open Session+Store → fetch UNSEEN → deliver → mark SEEN → close). The default provider reads a single account from MP Config `@ConfigProperty`. Content extraction is recursive to handle `multipart/mixed` envelopes.

**Tech Stack:** Java 21, Quarkus 3.32.2, Jakarta Mail via `org.eclipse.angus:angus-mail:2.0.3`, Greenmail for tests, JUnit 5, AssertJ.

**Spec:** `docs/specs/2026-05-29-email-inbound-connector-design.md` (rev 4)

---

## File Map

**New module `email-inbound/`:**
```
email-inbound/pom.xml
email-inbound/src/main/java/io/casehub/connectors/email/inbound/
  EmailInboundAccount.java           — record (value type, immutable)
  EmailInboundAccountProvider.java   — SPI interface (@FunctionalInterface)
  DefaultEmailInboundAccountProvider.java — @DefaultBean @ConfigProperty impl
  ContentExtractor.java              — package-private recursive MIME helper
  EmailInboundConnector.java         — InboundConnector impl, orchestrates polling

email-inbound/src/test/java/io/casehub/connectors/email/inbound/
  DefaultEmailInboundAccountProviderTest.java  — pure unit test
  ContentExtractorTest.java                    — pure unit test
  EmailInboundConnectorTest.java               — Greenmail unit tests (no Quarkus)
  GreenMailResource.java                       — QuarkusTestResourceLifecycleManager
  InboundMessageCapture.java                   — test CDI observer bean
  EmailInboundConnectorQuarkusTest.java        — @QuarkusTest integration test
```

**Modified:**
```
pom.xml — add <module>email-inbound</module>
```

---

## Task 1: Module Scaffold

**Files:**
- Modify: `pom.xml` (root)
- Create: `email-inbound/pom.xml`

- [ ] **Step 1: Add the module to the parent pom**

In `pom.xml`, add `<module>email-inbound</module>` to the `<modules>` block:

```xml
<modules>
  <module>core</module>
  <module>webhook</module>
  <module>email</module>
  <module>email-inbound</module>
</modules>
```

- [ ] **Step 2: Create email-inbound/pom.xml**

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 https://maven.apache.org/xsd/maven-4.0.0.xsd">
  <modelVersion>4.0.0</modelVersion>

  <parent>
    <groupId>io.casehub</groupId>
    <artifactId>casehub-connectors-parent</artifactId>
    <version>0.2-SNAPSHOT</version>
  </parent>

  <artifactId>casehub-connectors-email-inbound</artifactId>
  <name>CaseHub Connectors — Email Inbound</name>
  <description>IMAP polling inbound connector. Separate from casehub-connectors-email
(outbound/quarkus-mailer) because angus-mail and quarkus-mailer have no shared
infrastructure — keeping them together would force each on users who need only one.</description>

  <dependencies>
    <dependency>
      <groupId>io.casehub</groupId>
      <artifactId>casehub-connectors-core</artifactId>
      <version>${project.version}</version>
    </dependency>
    <dependency>
      <groupId>org.eclipse.angus</groupId>
      <artifactId>angus-mail</artifactId>
      <version>2.0.3</version>
    </dependency>

    <!-- Testing -->
    <dependency>
      <groupId>io.quarkus</groupId>
      <artifactId>quarkus-junit5</artifactId>
      <scope>test</scope>
    </dependency>
    <dependency>
      <groupId>com.icegreen</groupId>
      <artifactId>greenmail-junit5</artifactId>
      <version>2.1.3</version>
      <scope>test</scope>
    </dependency>
    <dependency>
      <groupId>org.awaitility</groupId>
      <artifactId>awaitility</artifactId>
      <scope>test</scope>
    </dependency>
    <dependency>
      <groupId>org.assertj</groupId>
      <artifactId>assertj-core</artifactId>
      <scope>test</scope>
    </dependency>
  </dependencies>

  <build>
    <plugins>
      <plugin>
        <groupId>io.smallrye</groupId>
        <artifactId>jandex-maven-plugin</artifactId>
        <version>3.3.1</version>
        <executions>
          <execution>
            <id>jandex</id>
            <phase>process-classes</phase>
            <goals><goal>jandex</goal></goals>
          </execution>
        </executions>
      </plugin>
      <plugin>
        <groupId>io.quarkus</groupId>
        <artifactId>quarkus-maven-plugin</artifactId>
        <version>${quarkus.version}</version>
        <extensions>true</extensions>
        <executions>
          <execution>
            <goals>
              <goal>build</goal>
              <goal>generate-code</goal>
              <goal>generate-code-tests</goal>
            </goals>
          </execution>
        </executions>
      </plugin>
    </plugins>
  </build>

</project>
```

- [ ] **Step 3: Verify the module resolves**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn -pl email-inbound validate
```

Expected: `BUILD SUCCESS`. If angus-mail or greenmail version not found, check Maven Central for the latest `org.eclipse.angus:angus-mail` and `com.icegreen:greenmail-junit5` and update accordingly.

---

## Task 2: EmailInboundAccount + EmailInboundAccountProvider

**Files:**
- Create: `email-inbound/src/main/java/io/casehub/connectors/email/inbound/EmailInboundAccount.java`
- Create: `email-inbound/src/main/java/io/casehub/connectors/email/inbound/EmailInboundAccountProvider.java`

These are pure types with no behaviour — no tests at this stage; they'll be exercised by all subsequent tasks.

- [ ] **Step 1: Create EmailInboundAccount**

```java
package io.casehub.connectors.email.inbound;

/**
 * Configuration for one IMAP account to poll.
 *
 * <p>{@code id} appears in {@code InboundMessage.metadata["account-id"]}
 * (not in {@code connectorId} — that is always {@code "email-inbound"}).
 */
public record EmailInboundAccount(
        String id,
        String host,
        int port,
        boolean tls,
        String username,
        String password,
        String folder,
        int pollIntervalSeconds) {
}
```

- [ ] **Step 2: Create EmailInboundAccountProvider**

```java
package io.casehub.connectors.email.inbound;

import java.util.List;

/**
 * SPI — returns the IMAP accounts to poll.
 *
 * <p>Override {@link DefaultEmailInboundAccountProvider} by providing an
 * {@code @ApplicationScoped} bean with higher CDI priority. The default
 * implementation reads a single account from MP Config.
 */
@FunctionalInterface
public interface EmailInboundAccountProvider {
    List<EmailInboundAccount> accounts();
}
```

- [ ] **Step 3: Compile**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn -pl email-inbound compile
```

Expected: `BUILD SUCCESS`

---

## Task 3: DefaultEmailInboundAccountProvider + Unit Tests

**Files:**
- Create: `email-inbound/src/main/java/io/casehub/connectors/email/inbound/DefaultEmailInboundAccountProvider.java`
- Create: `email-inbound/src/test/java/io/casehub/connectors/email/inbound/DefaultEmailInboundAccountProviderTest.java`

- [ ] **Step 1: Write the failing tests**

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
        assertThat(account.id()).isEqualTo("email-inbound");
        assertThat(account.host()).isEqualTo("imap.example.com");
        assertThat(account.port()).isEqualTo(993);
        assertThat(account.tls()).isTrue();
        assertThat(account.username()).isEqualTo("user@example.com");
        assertThat(account.password()).isEqualTo("secret");
        assertThat(account.folder()).isEqualTo("INBOX");
        assertThat(account.pollIntervalSeconds()).isEqualTo(60);
    }

    @Test
    void customFolder_preservedInAccount() {
        final DefaultEmailInboundAccountProvider provider =
                new DefaultEmailInboundAccountProvider(
                        "imap.example.com", 143, false, "user", "pass", "Support", 30);
        assertThat(provider.accounts().get(0).folder()).isEqualTo("Support");
        assertThat(provider.accounts().get(0).tls()).isFalse();
        assertThat(provider.accounts().get(0).pollIntervalSeconds()).isEqualTo(30);
    }
}
```

- [ ] **Step 2: Run tests — confirm they fail**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn -pl email-inbound test -Dtest=DefaultEmailInboundAccountProviderTest
```

Expected: compilation error — `DefaultEmailInboundAccountProvider` doesn't exist yet.

- [ ] **Step 3: Implement DefaultEmailInboundAccountProvider**

```java
package io.casehub.connectors.email.inbound;

import java.util.List;

import jakarta.enterprise.context.ApplicationScoped;
import jakarta.inject.Inject;

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

    @ConfigProperty(name = "casehub.connectors.email-inbound.poll-interval-seconds", defaultValue = "60")
    int pollIntervalSeconds;

    DefaultEmailInboundAccountProvider() {}

    DefaultEmailInboundAccountProvider(final String host, final int port, final boolean tls,
                                       final String username, final String password,
                                       final String folder, final int pollIntervalSeconds) {
        this.host = host;
        this.port = port;
        this.tls = tls;
        this.username = username;
        this.password = password;
        this.folder = folder;
        this.pollIntervalSeconds = pollIntervalSeconds;
    }

    @Override
    public List<EmailInboundAccount> accounts() {
        if (host == null || host.isBlank()) {
            return List.of();
        }
        return List.of(new EmailInboundAccount(
                "email-inbound", host, port, tls, username, password, folder, pollIntervalSeconds));
    }
}
```

- [ ] **Step 4: Run tests — confirm they pass**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn -pl email-inbound test -Dtest=DefaultEmailInboundAccountProviderTest
```

Expected: `Tests run: 4, Failures: 0, Errors: 0`

---

## Task 4: ContentExtractor + Unit Tests

**Files:**
- Create: `email-inbound/src/main/java/io/casehub/connectors/email/inbound/ContentExtractor.java`
- Create: `email-inbound/src/test/java/io/casehub/connectors/email/inbound/ContentExtractorTest.java`

- [ ] **Step 1: Write the failing tests**

```java
package io.casehub.connectors.email.inbound;

import static org.assertj.core.api.Assertions.assertThat;

import java.util.Properties;
import jakarta.mail.BodyPart;
import jakarta.mail.Message;
import jakarta.mail.Multipart;
import jakarta.mail.Part;
import jakarta.mail.Session;
import jakarta.mail.internet.MimeBodyPart;
import jakarta.mail.internet.MimeMessage;
import jakarta.mail.internet.MimeMultipart;

import org.junit.jupiter.api.Test;

class ContentExtractorTest {

    private static Session emptySession() {
        return Session.getInstance(new Properties());
    }

    @Test
    void plainText_returnedDirectly() throws Exception {
        final MimeMessage msg = new MimeMessage(emptySession());
        msg.setText("Hello world", "UTF-8");
        assertThat(ContentExtractor.extractContent(msg)).isEqualTo("Hello world");
    }

    @Test
    void htmlOnly_returnedAsRawHtml() throws Exception {
        final MimeMessage msg = new MimeMessage(emptySession());
        msg.setContent("<p>Hello</p>", "text/html; charset=UTF-8");
        assertThat(ContentExtractor.extractContent(msg)).isEqualTo("<p>Hello</p>");
    }

    @Test
    void multipartAlternative_preferPlainText() throws Exception {
        final MimeMessage msg = new MimeMessage(emptySession());
        final MimeMultipart mp = new MimeMultipart("alternative");

        final MimeBodyPart plain = new MimeBodyPart();
        plain.setText("Plain text body", "UTF-8");

        final MimeBodyPart html = new MimeBodyPart();
        html.setContent("<p>HTML body</p>", "text/html; charset=UTF-8");

        mp.addBodyPart(plain);
        mp.addBodyPart(html);
        msg.setContent(mp);

        assertThat(ContentExtractor.extractContent(msg)).isEqualTo("Plain text body");
    }

    @Test
    void multipartMixedWithNestedAlternative_extractsPlainText() throws Exception {
        // Structure: multipart/mixed
        //   └── multipart/alternative
        //         ├── text/plain  ← want this
        //         └── text/html
        //   └── application/pdf  ← ignored
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

        final MimeBodyPart attachment = new MimeBodyPart();
        attachment.setContent("pdf bytes", "application/pdf");
        attachment.setDisposition(Part.ATTACHMENT);
        attachment.setFileName("report.pdf");
        mixed.addBodyPart(attachment);

        msg.setContent(mixed);

        assertThat(ContentExtractor.extractContent(msg)).isEqualTo("Plain body");
    }

    @Test
    void binaryOnly_returnsEmptyString() throws Exception {
        final MimeMessage msg = new MimeMessage(emptySession());
        msg.setContent("bytes", "application/octet-stream");
        assertThat(ContentExtractor.extractContent(msg)).isEmpty();
    }
}
```

- [ ] **Step 2: Run tests — confirm they fail**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn -pl email-inbound test -Dtest=ContentExtractorTest
```

Expected: compilation error — `ContentExtractor` doesn't exist yet.

- [ ] **Step 3: Implement ContentExtractor**

```java
package io.casehub.connectors.email.inbound;

import java.io.IOException;
import java.util.logging.Level;
import java.util.logging.Logger;

import jakarta.mail.MessagingException;
import jakarta.mail.Multipart;
import jakarta.mail.Part;

/**
 * Recursive MIME content extractor. Prefers {@code text/plain}; falls back to
 * {@code text/html}; returns {@code ""} for binary-only messages.
 */
final class ContentExtractor {

    private static final Logger LOG = Logger.getLogger(ContentExtractor.class.getName());

    private ContentExtractor() {}

    static String extractContent(final Part part) {
        try {
            final String text = extractText(part);
            if (text != null) return text;
            final String html = extractHtml(part);
            if (html != null) return html;
        } catch (final Exception e) {
            LOG.log(Level.WARNING, "email-inbound: content extraction failed", e);
        }
        return "";
    }

    private static String extractText(final Part part) throws MessagingException, IOException {
        if (part.isMimeType("text/plain")) {
            return part.getContent().toString();
        }
        if (part.isMimeType("multipart/*")) {
            final Multipart mp = (Multipart) part.getContent();
            for (int i = 0; i < mp.getCount(); i++) {
                final String result = extractText(mp.getBodyPart(i));
                if (result != null) return result;
            }
        }
        return null;
    }

    private static String extractHtml(final Part part) throws MessagingException, IOException {
        if (part.isMimeType("text/html")) {
            return part.getContent().toString();
        }
        if (part.isMimeType("multipart/*")) {
            final Multipart mp = (Multipart) part.getContent();
            for (int i = 0; i < mp.getCount(); i++) {
                final String result = extractHtml(mp.getBodyPart(i));
                if (result != null) return result;
            }
        }
        return null;
    }
}
```

- [ ] **Step 4: Run tests — confirm they pass**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn -pl email-inbound test -Dtest=ContentExtractorTest
```

Expected: `Tests run: 5, Failures: 0, Errors: 0`

---

## Task 5: EmailInboundConnector Scaffold + No-Accounts Test

**Files:**
- Create: `email-inbound/src/main/java/io/casehub/connectors/email/inbound/EmailInboundConnector.java`
- Create: `email-inbound/src/test/java/io/casehub/connectors/email/inbound/EmailInboundConnectorTest.java` (initial)

- [ ] **Step 1: Write the no-accounts failing test**

```java
package io.casehub.connectors.email.inbound;

import static org.assertj.core.api.Assertions.assertThat;

import java.time.Instant;
import java.util.ArrayList;
import java.util.Date;
import java.util.List;
import java.util.Properties;

import jakarta.mail.Message;
import jakarta.mail.Session;
import jakarta.mail.internet.InternetAddress;
import jakarta.mail.internet.MimeMessage;

import com.icegreen.greenmail.configuration.GreenMailConfiguration;
import com.icegreen.greenmail.junit5.GreenMailExtension;
import com.icegreen.greenmail.util.ServerSetupTest;
import io.casehub.connectors.InboundMessage;

import org.junit.jupiter.api.AfterEach;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;
import org.junit.jupiter.api.extension.RegisterExtension;

class EmailInboundConnectorTest {

    @RegisterExtension
    static final GreenMailExtension GREEN_MAIL = new GreenMailExtension(ServerSetupTest.IMAP)
            .withConfiguration(GreenMailConfiguration.aConfig()
                    .withUser("inbox@example.com", "password"))
            .withPerMethodLifecycle(false);

    private List<InboundMessage> captured;
    private EmailInboundConnector connector;

    @BeforeEach
    void setUp() {
        captured = new ArrayList<>();
        connector = new EmailInboundConnector(() -> List.of(testAccount()));
    }

    @AfterEach
    void tearDown() {
        connector.stop();
    }

    private EmailInboundAccount testAccount() {
        return new EmailInboundAccount(
                "email-inbound",
                "localhost",
                GREEN_MAIL.getImap().getPort(),
                false,
                "inbox@example.com",
                "password",
                "INBOX",
                60);
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
        GREEN_MAIL.deliver(msg);
    }

    // ── no-accounts ──────────────────────────────────────────────────────────

    @Test
    void noAccounts_startIsNoOp_stopIsNoOp() {
        final EmailInboundConnector empty = new EmailInboundConnector(List::of);
        empty.start(captured::add);
        empty.stop(); // must not throw even if start was never called with accounts
        assertThat(captured).isEmpty();
    }

    @Test
    void id_returnsEmailInbound() {
        assertThat(connector.id()).isEqualTo("email-inbound");
    }
}
```

- [ ] **Step 2: Run tests — confirm they fail**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn -pl email-inbound test -Dtest=EmailInboundConnectorTest
```

Expected: compilation error — `EmailInboundConnector` doesn't exist yet.

- [ ] **Step 3: Implement EmailInboundConnector scaffold**

```java
package io.casehub.connectors.email.inbound;

import java.time.Instant;
import java.util.ArrayList;
import java.util.Collections;
import java.util.LinkedHashMap;
import java.util.List;
import java.util.Map;
import java.util.Properties;
import java.util.concurrent.Executors;
import java.util.concurrent.ScheduledExecutorService;
import java.util.concurrent.TimeUnit;
import java.util.logging.Level;
import java.util.logging.Logger;

import jakarta.enterprise.context.ApplicationScoped;
import jakarta.inject.Inject;
import jakarta.mail.Address;
import jakarta.mail.Flags;
import jakarta.mail.Folder;
import jakarta.mail.Message;
import jakarta.mail.Session;
import jakarta.mail.Store;
import jakarta.mail.internet.InternetAddress;
import jakarta.mail.search.FlagTerm;

import io.casehub.connectors.InboundConnector;
import io.casehub.connectors.InboundMessage;
import io.casehub.connectors.InboundMessageSink;

/**
 * Pull-based inbound connector for IMAP mailboxes.
 *
 * <p>Polls every configured account on a per-account {@link ScheduledExecutorService}
 * (single-threaded, daemon). Each poll cycle opens a fresh {@link Session} and
 * {@link Store} — no reconnect logic, no persistent connection.
 *
 * <p>{@code connectorId} is always {@value #ID}. Per-account identity is in
 * {@code InboundMessage.metadata["account-id"]}.
 *
 * <h2>Delivery guarantee</h2>
 * At-least-once. If shutdown interrupts between {@code sink.receive()} and the
 * SEEN flag write, the message will be redelivered on next startup. Observers must
 * be idempotent.
 */
@ApplicationScoped
public class EmailInboundConnector implements InboundConnector {

    static final String ID = "email-inbound";

    private static final Logger LOG = Logger.getLogger(EmailInboundConnector.class.getName());

    private final EmailInboundAccountProvider provider;
    private final List<ScheduledExecutorService> executors = new ArrayList<>();

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
        for (final EmailInboundAccount account : provider.accounts()) {
            final ScheduledExecutorService executor = Executors.newSingleThreadScheduledExecutor(r -> {
                final Thread t = new Thread(r, "email-inbound-" + account.id() + "-poller");
                t.setDaemon(true);
                return t;
            });
            executor.scheduleWithFixedDelay(
                    () -> pollAccount(account, sink),
                    0L,
                    account.pollIntervalSeconds(),
                    TimeUnit.SECONDS);
            executors.add(executor);
        }
    }

    @Override
    public void stop() {
        executors.forEach(ScheduledExecutorService::shutdownNow);
    }

    // Package-private for unit tests — tests call this directly rather than
    // going through the executor.
    void pollAccount(final EmailInboundAccount account, final InboundMessageSink sink) {
        final Properties props = buildProperties(account);
        final Session session = Session.getInstance(props);
        Store store = null;
        Folder folder = null;
        try {
            store = session.getStore();
            store.connect(account.host(), account.username(), account.password());
            folder = store.getFolder(account.folder());
            folder.open(Folder.READ_WRITE);

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
            LOG.log(Level.WARNING, "email-inbound: poll failed for account "
                    + account.id() + ": " + e.getMessage(), e);
        } finally {
            closeQuietly(folder, store);
        }
    }

    private static Properties buildProperties(final EmailInboundAccount account) {
        final Properties props = new Properties();
        if (account.tls()) {
            props.put("mail.store.protocol", "imaps");
            props.put("mail.imaps.host", account.host());
            props.put("mail.imaps.port", String.valueOf(account.port()));
            props.put("mail.imaps.ssl.enable", "true");
        } else {
            props.put("mail.store.protocol", "imap");
            props.put("mail.imap.host", account.host());
            props.put("mail.imap.port", String.valueOf(account.port()));
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
                    buildMetadata(account, msg));
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
            final java.util.Date received = msg.getReceivedDate();
            if (received != null) return received.toInstant();
            final java.util.Date sent = msg.getSentDate();
            if (sent != null) return sent.toInstant();
        } catch (final Exception ignored) {}
        return Instant.now();
    }

    private static Map<String, String> buildMetadata(final EmailInboundAccount account,
                                                     final Message msg) {
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
        return Collections.unmodifiableMap(meta);
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

- [ ] **Step 4: Run tests — confirm they pass**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn -pl email-inbound test -Dtest=EmailInboundConnectorTest
```

Expected: `Tests run: 2, Failures: 0, Errors: 0`

---

## Task 6: Poll Cycle — Delivery Tests

**Files:**
- Modify: `email-inbound/src/test/java/io/casehub/connectors/email/inbound/EmailInboundConnectorTest.java`

Add these tests to the existing `EmailInboundConnectorTest` class.

- [ ] **Step 1: Add delivery tests**

Add the following test methods to `EmailInboundConnectorTest` (inside the class, after the existing tests):

```java
    // ── delivery ─────────────────────────────────────────────────────────────

    @Test
    void noUnseenMessages_sinkNotCalled() throws Exception {
        connector.pollAccount(testAccount(), captured::add);
        assertThat(captured).isEmpty();
    }

    @Test
    void singlePlainTextMessage_deliveredWithCorrectFields() throws Exception {
        deliver("sender@example.com", "Hello subject", "Hello body");

        connector.pollAccount(testAccount(), captured::add);

        assertThat(captured).hasSize(1);
        final InboundMessage msg = captured.get(0);
        assertThat(msg.connectorId()).isEqualTo("email-inbound");
        assertThat(msg.externalSenderId()).isEqualTo("sender@example.com");
        assertThat(msg.externalChannelRef()).isEqualTo("inbox@example.com");
        assertThat(msg.content()).isEqualTo("Hello body");
        assertThat(msg.receivedAt()).isNotNull();
        assertThat(msg.metadata()).containsEntry("account-id", "email-inbound");
        assertThat(msg.metadata()).containsEntry("subject", "Hello subject");
        assertThat(msg.metadata()).containsKey("message-id");
    }

    @Test
    void multipleUnseenMessages_allDeliveredAndMarkedSeen() throws Exception {
        deliver("a@example.com", "First", "Body A");
        deliver("b@example.com", "Second", "Body B");

        connector.pollAccount(testAccount(), captured::add);

        assertThat(captured).hasSize(2);
        assertThat(captured).extracting(InboundMessage::content)
                .containsExactlyInAnyOrder("Body A", "Body B");
    }

    @Test
    void secondPoll_alreadySeenNotRedelivered() throws Exception {
        deliver("sender@example.com", "Once", "Only once");

        connector.pollAccount(testAccount(), captured::add);
        assertThat(captured).hasSize(1);

        captured.clear();
        connector.pollAccount(testAccount(), captured::add);
        assertThat(captured).isEmpty();
    }

    @Test
    void htmlOnlyMessage_rawHtmlInContent() throws Exception {
        final MimeMessage msg = new MimeMessage(Session.getInstance(new Properties()));
        msg.setFrom(new InternetAddress("sender@example.com"));
        msg.setRecipient(Message.RecipientType.TO, new InternetAddress("inbox@example.com"));
        msg.setSubject("HTML email");
        msg.setContent("<p>Rich content</p>", "text/html; charset=UTF-8");
        msg.setSentDate(Date.from(Instant.now()));
        GREEN_MAIL.deliver(msg);

        connector.pollAccount(testAccount(), captured::add);

        assertThat(captured).hasSize(1);
        assertThat(captured.get(0).content()).isEqualTo("<p>Rich content</p>");
    }
```

- [ ] **Step 2: Run tests**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn -pl email-inbound test -Dtest=EmailInboundConnectorTest
```

Expected: `Tests run: 7, Failures: 0, Errors: 0`

- [ ] **Step 3: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/connectors add email-inbound/ pom.xml
git -C /Users/mdproctor/claude/casehub/connectors commit -m "feat(email-inbound): scaffold module, types, ContentExtractor, EmailInboundConnector, delivery tests — refs #7"
```

---

## Task 7: Poll Cycle — Edge Cases

**Files:**
- Modify: `email-inbound/src/test/java/io/casehub/connectors/email/inbound/EmailInboundConnectorTest.java`

Add these tests to the existing class.

- [ ] **Step 1: Add edge-case tests**

```java
    // ── edge cases ───────────────────────────────────────────────────────────

    @Test
    void sinkThrows_messageStillMarkedSeen_remainingDelivered() throws Exception {
        deliver("a@example.com", "First", "Body A");
        deliver("b@example.com", "Second", "Body B");

        final List<String> contents = new ArrayList<>();
        // Throw on the first message only
        final boolean[] first = {true};
        connector.pollAccount(testAccount(), msg -> {
            contents.add(msg.content());
            if (first[0]) {
                first[0] = false;
                throw new RuntimeException("Sink error");
            }
        });

        // Both messages should have been attempted
        assertThat(contents).hasSize(2);

        // Both must be SEEN (no redelivery on next poll)
        captured.clear();
        connector.pollAccount(testAccount(), captured::add);
        assertThat(captured).isEmpty();
    }

    @Test
    void imapConnectionFailure_loggedAndNoSinkCall() {
        final EmailInboundAccount badAccount = new EmailInboundAccount(
                "email-inbound", "localhost", 19999, false,
                "user", "pass", "INBOX", 60);

        // Must not throw — exception is logged internally
        connector.pollAccount(badAccount, captured::add);
        assertThat(captured).isEmpty();
    }

    @Test
    void missingFromHeader_senderIdIsEmptyString() throws Exception {
        // Deliver a message with no From header
        final MimeMessage msg = new MimeMessage(Session.getInstance(new Properties()));
        msg.setRecipient(Message.RecipientType.TO, new InternetAddress("inbox@example.com"));
        msg.setSubject("No from");
        msg.setText("Body");
        msg.setSentDate(Date.from(Instant.now()));
        GREEN_MAIL.deliver(msg);

        connector.pollAccount(testAccount(), captured::add);

        assertThat(captured).hasSize(1);
        assertThat(captured.get(0).externalSenderId()).isEmpty();
    }

    @Test
    void missingToHeader_channelRefFallsBackToAccountUsername() throws Exception {
        // Deliver directly without a To header (BCC-style)
        final MimeMessage msg = new MimeMessage(Session.getInstance(new Properties()));
        msg.setFrom(new InternetAddress("sender@example.com"));
        msg.setSubject("No To");
        msg.setText("BCC body");
        msg.setSentDate(Date.from(Instant.now()));
        GREEN_MAIL.deliver(msg);

        connector.pollAccount(testAccount(), captured::add);

        assertThat(captured).hasSize(1);
        assertThat(captured.get(0).externalChannelRef()).isEqualTo("inbox@example.com");
    }

    @Test
    void messageWithoutMessageIdHeader_metadataKeyAbsent() throws Exception {
        final MimeMessage msg = new MimeMessage(Session.getInstance(new Properties()));
        msg.setFrom(new InternetAddress("sender@example.com"));
        msg.setRecipient(Message.RecipientType.TO, new InternetAddress("inbox@example.com"));
        msg.setSubject("No message-id");
        msg.setText("Body");
        // Do NOT set Message-ID header
        GREEN_MAIL.deliver(msg);

        connector.pollAccount(testAccount(), captured::add);

        assertThat(captured).hasSize(1);
        // account-id always present; message-id absent when header missing
        assertThat(captured.get(0).metadata()).containsKey("account-id");
        assertThat(captured.get(0).metadata()).doesNotContainKey("message-id");
    }

    @Test
    void messageWithoutSubject_subjectKeyAbsent() throws Exception {
        final MimeMessage msg = new MimeMessage(Session.getInstance(new Properties()));
        msg.setFrom(new InternetAddress("sender@example.com"));
        msg.setRecipient(Message.RecipientType.TO, new InternetAddress("inbox@example.com"));
        msg.setText("Body");
        msg.setSentDate(Date.from(Instant.now()));
        GREEN_MAIL.deliver(msg);

        connector.pollAccount(testAccount(), captured::add);

        assertThat(captured).hasSize(1);
        assertThat(captured.get(0).metadata()).doesNotContainKey("subject");
    }
```

- [ ] **Step 2: Run all EmailInboundConnector tests**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn -pl email-inbound test -Dtest=EmailInboundConnectorTest
```

Expected: `Tests run: 13, Failures: 0, Errors: 0`

- [ ] **Step 3: Run all module tests so far**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn -pl email-inbound test
```

Expected: all pass (DefaultEmailInboundAccountProviderTest + ContentExtractorTest + EmailInboundConnectorTest)

- [ ] **Step 4: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/connectors add email-inbound/
git -C /Users/mdproctor/claude/casehub/connectors commit -m "test(email-inbound): edge cases — sink throws, connection failure, header fallbacks — refs #7"
```

---

## Task 8: Integration Test (@QuarkusTest)

**Files:**
- Create: `email-inbound/src/test/java/io/casehub/connectors/email/inbound/GreenMailResource.java`
- Create: `email-inbound/src/test/java/io/casehub/connectors/email/inbound/InboundMessageCapture.java`
- Create: `email-inbound/src/test/java/io/casehub/connectors/email/inbound/EmailInboundConnectorQuarkusTest.java`

- [ ] **Step 1: Create GreenMailResource**

```java
package io.casehub.connectors.email.inbound;

import java.util.Map;

import com.icegreen.greenmail.configuration.GreenMailConfiguration;
import com.icegreen.greenmail.util.GreenMail;
import com.icegreen.greenmail.util.ServerSetupTest;
import io.quarkus.test.common.QuarkusTestResourceLifecycleManager;

public class GreenMailResource implements QuarkusTestResourceLifecycleManager {

    private GreenMail greenMail;

    @Override
    public Map<String, String> start() {
        greenMail = new GreenMail(ServerSetupTest.IMAP);
        greenMail.withConfiguration(GreenMailConfiguration.aConfig()
                .withUser("inbox@example.com", "password"));
        greenMail.start();
        return Map.of(
                "casehub.connectors.email-inbound.host", "localhost",
                "casehub.connectors.email-inbound.port",
                        String.valueOf(greenMail.getImap().getPort()),
                "casehub.connectors.email-inbound.tls", "false",
                "casehub.connectors.email-inbound.username", "inbox@example.com",
                "casehub.connectors.email-inbound.password", "password",
                "casehub.connectors.email-inbound.poll-interval-seconds", "1");
    }

    @Override
    public void stop() {
        if (greenMail != null) greenMail.stop();
    }

    GreenMail greenMail() {
        return greenMail;
    }
}
```

- [ ] **Step 2: Create InboundMessageCapture**

```java
package io.casehub.connectors.email.inbound;

import java.util.ArrayList;
import java.util.Collections;
import java.util.List;

import jakarta.enterprise.context.ApplicationScoped;
import jakarta.enterprise.event.Observes;

import io.casehub.connectors.InboundMessage;

@ApplicationScoped
public class InboundMessageCapture {

    private final List<InboundMessage> messages = Collections.synchronizedList(new ArrayList<>());

    void observe(@Observes final InboundMessage message) {
        messages.add(message);
    }

    public List<InboundMessage> messages() {
        return Collections.unmodifiableList(messages);
    }

    public void clear() {
        messages.clear();
    }
}
```

- [ ] **Step 3: Write the failing integration test**

```java
package io.casehub.connectors.email.inbound;

import static org.assertj.core.api.Assertions.assertThat;
import static org.awaitility.Awaitility.await;

import java.time.Duration;
import java.time.Instant;
import java.util.Date;
import java.util.Properties;

import jakarta.inject.Inject;
import jakarta.mail.Message;
import jakarta.mail.Session;
import jakarta.mail.internet.InternetAddress;
import jakarta.mail.internet.MimeMessage;

import io.casehub.connectors.InboundMessage;
import io.quarkus.test.common.QuarkusTestResource;
import io.quarkus.test.junit.QuarkusTest;

import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

@QuarkusTest
@QuarkusTestResource(GreenMailResource.class)
class EmailInboundConnectorQuarkusTest {

    @Inject
    InboundMessageCapture capture;

    @InjectMock(convertScopes = false) // not needed — use resource handle
    GreenMailResource resource; // won't work this way — see below

    // We need access to the GreenMail instance from GreenMailResource.
    // Use @InjectMock is wrong here. Use the static reference approach:
    // GreenMailResource injects itself via @InjectResource — but Quarkus
    // doesn't do that. Instead, store it as a static field in the resource.
```

Hmm — accessing the GreenMail instance inside `@QuarkusTest` requires a different approach. The cleanest solution is to store the GreenMail instance statically in a holder class:

Replace the integration test with this corrected version:

- [ ] **Step 3 (revised): Correct integration test**

Update `GreenMailResource` to expose the GreenMail instance statically:

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
        INSTANCE = new GreenMail(ServerSetupTest.IMAP);
        INSTANCE.withConfiguration(GreenMailConfiguration.aConfig()
                .withUser("inbox@example.com", "password"));
        INSTANCE.start();
        return Map.of(
                "casehub.connectors.email-inbound.host", "localhost",
                "casehub.connectors.email-inbound.port",
                        String.valueOf(INSTANCE.getImap().getPort()),
                "casehub.connectors.email-inbound.tls", "false",
                "casehub.connectors.email-inbound.username", "inbox@example.com",
                "casehub.connectors.email-inbound.password", "password",
                "casehub.connectors.email-inbound.poll-interval-seconds", "1");
    }

    @Override
    public void stop() {
        if (INSTANCE != null) INSTANCE.stop();
    }
}
```

And write the integration test:

```java
package io.casehub.connectors.email.inbound;

import static org.assertj.core.api.Assertions.assertThat;
import static org.awaitility.Awaitility.await;

import java.time.Duration;
import java.time.Instant;
import java.util.Date;
import java.util.Properties;

import jakarta.inject.Inject;
import jakarta.mail.Message;
import jakarta.mail.Session;
import jakarta.mail.internet.InternetAddress;
import jakarta.mail.internet.MimeMessage;

import io.casehub.connectors.InboundMessage;
import io.quarkus.test.common.QuarkusTestResource;
import io.quarkus.test.junit.QuarkusTest;

import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

@QuarkusTest
@QuarkusTestResource(GreenMailResource.class)
class EmailInboundConnectorQuarkusTest {

    @Inject
    InboundMessageCapture capture;

    @BeforeEach
    void clearCapture() {
        capture.clear();
    }

    @Test
    void happyPath_emailDelivered_cdiFired() throws Exception {
        // Deliver a plain-text email to the Greenmail IMAP inbox
        final MimeMessage msg = new MimeMessage(Session.getInstance(new Properties()));
        msg.setFrom(new InternetAddress("sender@example.com"));
        msg.setRecipient(Message.RecipientType.TO, new InternetAddress("inbox@example.com"));
        msg.setSubject("Integration test");
        msg.setText("Integration body");
        msg.setSentDate(Date.from(Instant.now()));
        msg.setHeader("Message-ID", "<integration-test@example.com>");
        GreenMailResource.INSTANCE.deliver(msg);

        // Poll interval is 1s — await CDI event delivery within 5s
        await().atMost(Duration.ofSeconds(5))
                .until(() -> !capture.messages().isEmpty());

        final InboundMessage delivered = capture.messages().get(0);
        assertThat(delivered.connectorId()).isEqualTo("email-inbound");
        assertThat(delivered.externalSenderId()).isEqualTo("sender@example.com");
        assertThat(delivered.externalChannelRef()).isEqualTo("inbox@example.com");
        assertThat(delivered.content()).isEqualTo("Integration body");
        assertThat(delivered.metadata()).containsEntry("account-id", "email-inbound");
        assertThat(delivered.metadata()).containsEntry("subject", "Integration test");
    }
}
```

- [ ] **Step 4: Run the integration test**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn -pl email-inbound test -Dtest=EmailInboundConnectorQuarkusTest
```

Expected: `Tests run: 1, Failures: 0, Errors: 0`

If the test times out waiting for the CDI event: check that `InboundConnectorService` is discovering `EmailInboundConnector` (it should via `@All List<InboundConnector>`). The Quarkus Jandex index must include both modules — verify Jandex plugin is in the pom.

- [ ] **Step 5: Run full module test suite**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn -pl email-inbound test
```

Expected: all tests pass.

- [ ] **Step 6: Run full project build**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn clean install
```

Expected: `BUILD SUCCESS`. All four modules build and all tests pass.

- [ ] **Step 7: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/connectors add email-inbound/ pom.xml
git -C /Users/mdproctor/claude/casehub/connectors commit -m "feat(email-inbound): integration test — QuarkusTest happy path — refs #7"
```

---

## Self-Review

**Spec coverage check:**

| Spec requirement | Task |
|---|---|
| `email-inbound` separate module | Task 1 |
| `EmailInboundAccount` record with all 8 fields | Task 2 |
| `EmailInboundAccountProvider` SPI | Task 2 |
| `DefaultEmailInboundAccountProvider` @DefaultBean @ConfigProperty | Task 3 |
| Blank host → empty list | Task 3 test |
| `ContentExtractor` recursive multipart | Task 4 |
| Plain/HTML/mixed extraction + binary → `""` | Task 4 tests |
| `EmailInboundConnector.id()` = `"email-inbound"` | Task 5 |
| Per-account ScheduledExecutorService, daemon threads, named | Task 5 impl |
| `scheduleWithFixedDelay` | Task 5 impl |
| Session+Store opened fresh per poll | Task 5 impl |
| UNSEEN search + SEEN flag after delivery | Task 5 impl |
| `connectorId` always `"email-inbound"` | Task 6 test |
| `externalSenderId` = `InternetAddress.getAddress()` | Task 6 test |
| `externalChannelRef` = `To:` address | Task 6 test |
| `receivedAt` fallback chain (getReceivedDate→getSentDate→now) | Task 5 impl + Task 7 could add explicit test |
| `metadata["account-id"]` always present | Task 6 test |
| `metadata["message-id"]` omitted when absent | Task 7 test |
| `metadata["subject"]` omitted when absent | Task 7 test |
| `sink.receive()` throws → SEEN still set | Task 7 test |
| IMAP connection failure → logged, no crash | Task 7 test |
| `From:` absent → `""` | Task 7 test |
| `To:` absent → `account.username()` | Task 7 test |
| Empty accounts → no-op start/stop | Task 5 test |
| At-least-once documented | Task 5 impl (javadoc) |
| Integration test: Greenmail + @QuarkusTest + CDI event | Task 8 |
| Test `tls=false` for both unit and integration | Tasks 5/8 |
| `poll-interval-seconds=1` in integration test profile | Task 8 GreenMailResource |

**Placeholder scan:** no TBD/TODO found.

**Type consistency:** `EmailInboundAccount` fields (id, host, port, tls, username, password, folder, pollIntervalSeconds) used consistently across all tasks. `ContentExtractor.extractContent(Part)` called in `EmailInboundConnector.toInboundMessage()`. `EmailInboundConnector.ID = "email-inbound"` used in both impl and test assertions.

**One gap identified:** receivedAt fallback chain has an explicit test for the first case (normal email with sent date) but no isolated test for the `getReceivedDate()→getSentDate()→now` chain. This is acceptable — the behaviour is covered implicitly in Task 6 tests (real emails through Greenmail) and the fallback for both null cases is simple enough not to warrant a separate Greenmail test.
