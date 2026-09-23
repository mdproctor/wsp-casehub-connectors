# BankFeedPlatform and EmailPlatform SPIs — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #94 — New SPIs — BankFeedPlatform and EmailPlatform with demo impls
**Issue group:** #94

**Goal:** Add two flat platform SPIs (BankFeedPlatform, EmailPlatform) with `@SimulationEligible` annotation, model records, platform services, and `@DefaultBean` no-op fallbacks.

**Architecture:** Two new Maven modules (`bank-spi`, `email-spi`) following the `calendar-spi` pattern exactly — flat interface, model records as Java records, CDI-produced platform service, `@DefaultBean` no-op. Both SPIs annotated with `@SimulationEligible` for build-time decorator generation by the platform simulation framework. Shared `Page<T>` and `PageRequest` pagination types added to `connectors-api`.

**Tech Stack:** Java 21 (on Java 26 JVM), Quarkus 3.32.2, simulation-api (platform dependency)

## Global Constraints

- All casehubio artifacts are `0.2-SNAPSHOT`
- SPI identifier methods are named `id()` (protocol PP-20260609-e3a2bd)
- Jandex plugin required in every module pom for CDI discovery
- `BigDecimal` for monetary amounts — never floating point
- Build: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn clean install`
- Tests use AssertJ assertions, JUnit 5

---

## Batch 1: BankFeedPlatform SPI

### Task 1: Shared pagination types in connectors-api

**Files:**
- Create: `connectors-api/src/main/java/io/casehub/connectors/PageRequest.java`
- Create: `connectors-api/src/main/java/io/casehub/connectors/Page.java`
- Test: `connectors-api/src/test/java/io/casehub/connectors/PageTest.java`

**Interfaces:**
- Consumes: nothing
- Produces: `Page<T>(List<T> items, String nextCursor, boolean hasMore)` with `Page.of(List<T>)` factory, `PageRequest(String cursor, int pageSize)` with `PageRequest.first(int)` factory

- [ ] **Step 1: Write failing tests for Page and PageRequest**

```java
package io.casehub.connectors;

import org.junit.jupiter.api.Test;
import java.util.List;
import static org.assertj.core.api.Assertions.assertThat;

class PageTest {

    @Test
    void pageOf_wrapsItemsWithNoMorePages() {
        var page = Page.of(List.of("a", "b"));
        assertThat(page.items()).containsExactly("a", "b");
        assertThat(page.nextCursor()).isNull();
        assertThat(page.hasMore()).isFalse();
    }

    @Test
    void page_withCursor_indicatesMorePages() {
        var page = new Page<>(List.of("a"), "cursor-2", true);
        assertThat(page.items()).containsExactly("a");
        assertThat(page.nextCursor()).isEqualTo("cursor-2");
        assertThat(page.hasMore()).isTrue();
    }

    @Test
    void pageRequestFirst_hasNullCursor() {
        var req = PageRequest.first(25);
        assertThat(req.cursor()).isNull();
        assertThat(req.pageSize()).isEqualTo(25);
    }

    @Test
    void pageRequest_withCursor() {
        var req = new PageRequest("abc", 50);
        assertThat(req.cursor()).isEqualTo("abc");
        assertThat(req.pageSize()).isEqualTo(50);
    }
}
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl connectors-api -Dtest=PageTest -f /Users/mdproctor/claude/casehub/connectors/pom.xml`
Expected: FAIL — classes not found

- [ ] **Step 3: Implement Page and PageRequest**

Create `connectors-api/src/main/java/io/casehub/connectors/PageRequest.java`:

```java
package io.casehub.connectors;

public record PageRequest(String cursor, int pageSize) {

    public static PageRequest first(int pageSize) {
        return new PageRequest(null, pageSize);
    }
}
```

Create `connectors-api/src/main/java/io/casehub/connectors/Page.java`:

```java
package io.casehub.connectors;

import java.util.List;

public record Page<T>(List<T> items, String nextCursor, boolean hasMore) {

    public static <T> Page<T> of(List<T> items) {
        return new Page<>(items, null, false);
    }
}
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl connectors-api -Dtest=PageTest -f /Users/mdproctor/claude/casehub/connectors/pom.xml`
Expected: PASS

- [ ] **Step 5: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/connectors add connectors-api/src/main/java/io/casehub/connectors/Page.java connectors-api/src/main/java/io/casehub/connectors/PageRequest.java connectors-api/src/test/java/io/casehub/connectors/PageTest.java
git -C /Users/mdproctor/claude/casehub/connectors commit -m "feat(api): add Page<T> and PageRequest shared pagination types Refs #94"
```

### Task 2: bank-spi module — SPI, models, service, no-op, tests

**Files:**
- Create: `bank-spi/pom.xml`
- Create: `bank-spi/src/main/java/io/casehub/connectors/bank/spi/BankFeedPlatform.java`
- Create: `bank-spi/src/main/java/io/casehub/connectors/bank/model/AccountInfo.java`
- Create: `bank-spi/src/main/java/io/casehub/connectors/bank/model/AccountType.java`
- Create: `bank-spi/src/main/java/io/casehub/connectors/bank/model/AccountBalance.java`
- Create: `bank-spi/src/main/java/io/casehub/connectors/bank/model/Transaction.java`
- Create: `bank-spi/src/main/java/io/casehub/connectors/bank/model/TransactionDirection.java`
- Create: `bank-spi/src/main/java/io/casehub/connectors/bank/model/TransactionStatus.java`
- Create: `bank-spi/src/main/java/io/casehub/connectors/bank/BankFeedPlatformService.java`
- Create: `bank-spi/src/main/java/io/casehub/connectors/bank/BankBeans.java`
- Create: `bank-spi/src/main/java/io/casehub/connectors/bank/NoOpBankFeedPlatform.java`
- Modify: `pom.xml` (parent — add `<module>bank-spi</module>`)
- Test: `bank-spi/src/test/java/io/casehub/connectors/bank/BankFeedPlatformServiceTest.java`

**Interfaces:**
- Consumes: `Page<T>`, `PageRequest` from connectors-api (Task 1)
- Produces: `BankPlatform` SPI interface, `BankFeedPlatformService.platform(id)`, all model records

- [ ] **Step 1: Create bank-spi/pom.xml and add to parent**

Create `bank-spi/pom.xml`:

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

  <artifactId>casehub-connectors-bank-spi</artifactId>
  <name>CaseHub Connectors — Bank Feed SPI</name>
  <description>Bank Feed Platform SPI: BankFeedPlatform interface, model types,
and BankFeedPlatformService routing.</description>

  <dependencies>
    <dependency>
      <groupId>io.casehub</groupId>
      <artifactId>casehub-connectors-api</artifactId>
      <version>${project.version}</version>
    </dependency>
    <dependency>
      <groupId>io.casehub</groupId>
      <artifactId>casehub-platform-simulation-api</artifactId>
      <version>${project.version}</version>
    </dependency>
    <dependency>
      <groupId>io.quarkus</groupId>
      <artifactId>quarkus-arc</artifactId>
    </dependency>

    <!-- Testing -->
    <dependency>
      <groupId>io.quarkus</groupId>
      <artifactId>quarkus-junit</artifactId>
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
    </plugins>
  </build>

</project>
```

Add `<module>bank-spi</module>` to the parent pom's `<modules>` section (after `calendar-google`).

- [ ] **Step 2: Create model records and enums**

Create `bank-spi/src/main/java/io/casehub/connectors/bank/model/AccountType.java`:

```java
package io.casehub.connectors.bank.model;

public enum AccountType { CURRENT, SAVINGS, CREDIT_CARD, LOAN, MORTGAGE, OTHER }
```

Create `bank-spi/src/main/java/io/casehub/connectors/bank/model/TransactionDirection.java`:

```java
package io.casehub.connectors.bank.model;

public enum TransactionDirection { DEBIT, CREDIT }
```

Create `bank-spi/src/main/java/io/casehub/connectors/bank/model/TransactionStatus.java`:

```java
package io.casehub.connectors.bank.model;

public enum TransactionStatus { PENDING, BOOKED }
```

Create `bank-spi/src/main/java/io/casehub/connectors/bank/model/AccountInfo.java`:

```java
package io.casehub.connectors.bank.model;

public record AccountInfo(String id, String name,
                          AccountType type, String currency) {}
```

Create `bank-spi/src/main/java/io/casehub/connectors/bank/model/AccountBalance.java`:

```java
package io.casehub.connectors.bank.model;

import java.math.BigDecimal;
import java.time.Instant;

public record AccountBalance(String accountId,
                             BigDecimal available, BigDecimal current,
                             String currency, Instant asOf) {}
```

Create `bank-spi/src/main/java/io/casehub/connectors/bank/model/Transaction.java`:

```java
package io.casehub.connectors.bank.model;

import java.math.BigDecimal;
import java.time.LocalDate;

public record Transaction(String id, String accountId,
                          BigDecimal amount, TransactionDirection direction,
                          String currency,
                          String description, String merchantName,
                          String category,
                          LocalDate date, TransactionStatus status) {}
```

- [ ] **Step 3: Create BankFeedPlatform SPI interface**

Create `bank-spi/src/main/java/io/casehub/connectors/bank/spi/BankFeedPlatform.java`:

```java
package io.casehub.connectors.bank.spi;

import java.time.Instant;
import java.util.List;

import io.casehub.connectors.Page;
import io.casehub.connectors.PageRequest;
import io.casehub.connectors.bank.model.AccountBalance;
import io.casehub.connectors.bank.model.AccountInfo;
import io.casehub.connectors.bank.model.Transaction;
import io.casehub.platform.simulation.SimulationEligible;

@SimulationEligible(name = "bank-feed-platform")
public interface BankFeedPlatform {

    String id();

    List<AccountInfo> listAccounts();

    AccountBalance balance(String accountId);

    Page<Transaction> listTransactions(String accountId,
                                       Instant from, Instant to,
                                       PageRequest pagination);

    Transaction getTransaction(String accountId, String transactionId);
}
```

- [ ] **Step 4: Write failing test for BankFeedPlatformService**

Create `bank-spi/src/test/java/io/casehub/connectors/bank/BankFeedPlatformServiceTest.java`:

```java
package io.casehub.connectors.bank;

import java.time.Instant;
import java.util.List;

import org.junit.jupiter.api.Test;

import io.casehub.connectors.Page;
import io.casehub.connectors.PageRequest;
import io.casehub.connectors.bank.model.AccountBalance;
import io.casehub.connectors.bank.model.AccountInfo;
import io.casehub.connectors.bank.model.Transaction;
import io.casehub.connectors.bank.spi.BankFeedPlatform;

import static org.assertj.core.api.Assertions.assertThat;
import static org.assertj.core.api.Assertions.assertThatThrownBy;

class BankFeedPlatformServiceTest {

    static class StubPlatform implements BankFeedPlatform {
        private final String platformId;
        StubPlatform(String id) { this.platformId = id; }
        @Override public String id() { return platformId; }
        @Override public List<AccountInfo> listAccounts() { return List.of(); }
        @Override public AccountBalance balance(String accountId) { return null; }
        @Override public Page<Transaction> listTransactions(String accountId,
                Instant from, Instant to, PageRequest pagination) { return Page.of(List.of()); }
        @Override public Transaction getTransaction(String accountId, String transactionId) { return null; }
    }

    @Test
    void platform_knownId_returnsPlatform() {
        var service = new BankFeedPlatformService(List.of(new StubPlatform("truelayer")));
        assertThat(service.platform("truelayer").id()).isEqualTo("truelayer");
    }

    @Test
    void platform_unknownId_throws() {
        var service = new BankFeedPlatformService(List.of(new StubPlatform("truelayer")));
        assertThatThrownBy(() -> service.platform("plaid"))
                .isInstanceOf(IllegalArgumentException.class)
                .hasMessageContaining("plaid")
                .hasMessageContaining("truelayer");
    }

    @Test
    void supports_knownId_returnsTrue() {
        var service = new BankFeedPlatformService(List.of(new StubPlatform("truelayer")));
        assertThat(service.supports("truelayer")).isTrue();
    }

    @Test
    void supports_unknownId_returnsFalse() {
        var service = new BankFeedPlatformService(List.of(new StubPlatform("truelayer")));
        assertThat(service.supports("plaid")).isFalse();
    }

    @Test
    void ids_returnsAllRegistered() {
        var service = new BankFeedPlatformService(
                List.of(new StubPlatform("truelayer"), new StubPlatform("plaid")));
        assertThat(service.ids()).containsExactlyInAnyOrder("truelayer", "plaid");
    }

    @Test
    void duplicateId_throwsAtConstruction() {
        assertThatThrownBy(() -> new BankFeedPlatformService(
                List.of(new StubPlatform("truelayer"), new StubPlatform("truelayer"))))
                .isInstanceOf(IllegalStateException.class)
                .hasMessageContaining("truelayer");
    }
}
```

- [ ] **Step 5: Run tests to verify they fail**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl bank-spi -Dtest=BankFeedPlatformServiceTest -f /Users/mdproctor/claude/casehub/connectors/pom.xml`
Expected: FAIL — `BankPlatformService` not found

- [ ] **Step 6: Implement BankFeedPlatformService, BankBeans, NoOpBankFeedPlatform**

Create `bank-spi/src/main/java/io/casehub/connectors/bank/BankFeedPlatformService.java`:

```java
package io.casehub.connectors.bank;

import java.util.List;
import java.util.Map;
import java.util.Set;
import java.util.function.Function;
import java.util.stream.Collectors;

import io.casehub.connectors.bank.spi.BankFeedPlatform;

public class BankFeedPlatformService {

    private final Map<String, BankFeedPlatform> registry;

    public BankFeedPlatformService(final List<BankFeedPlatform> platforms) {
        this.registry = platforms.stream()
                .collect(Collectors.toMap(
                        BankFeedPlatform::id,
                        Function.identity(),
                        (a, b) -> {
                            throw new IllegalStateException(
                                    "Duplicate bank feed platform id: '" + a.id() + "'");
                        }));
    }

    public BankFeedPlatform platform(final String id) {
        final BankFeedPlatform platform = registry.get(id);
        if (platform == null) {
            throw new IllegalArgumentException(
                    "No bank feed platform registered for id '" + id
                    + "'. Available: " + registry.keySet());
        }
        return platform;
    }

    public boolean supports(final String id) {
        return registry.containsKey(id);
    }

    public Set<String> ids() {
        return Set.copyOf(registry.keySet());
    }
}
```

Create `bank-spi/src/main/java/io/casehub/connectors/bank/BankBeans.java`:

```java
package io.casehub.connectors.bank;

import io.casehub.connectors.bank.spi.BankFeedPlatform;
import io.quarkus.arc.All;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.enterprise.inject.Produces;

import java.util.List;

@ApplicationScoped
public class BankBeans {

    @Produces
    @ApplicationScoped
    public BankFeedPlatformService bankFeedPlatformService(
            @All List<BankFeedPlatform> platforms) {
        return new BankFeedPlatformService(platforms);
    }
}
```

Create `bank-spi/src/main/java/io/casehub/connectors/bank/NoOpBankFeedPlatform.java`:

```java
package io.casehub.connectors.bank;

import java.time.Instant;
import java.util.List;

import io.casehub.connectors.Page;
import io.casehub.connectors.PageRequest;
import io.casehub.connectors.bank.model.AccountBalance;
import io.casehub.connectors.bank.model.AccountInfo;
import io.casehub.connectors.bank.model.Transaction;
import io.casehub.connectors.bank.spi.BankFeedPlatform;
import io.quarkus.arc.DefaultBean;
import jakarta.enterprise.context.ApplicationScoped;

@DefaultBean
@ApplicationScoped
public class NoOpBankFeedPlatform implements BankFeedPlatform {

    @Override
    public String id() {
        return "none";
    }

    @Override
    public List<AccountInfo> listAccounts() {
        return List.of();
    }

    @Override
    public AccountBalance balance(final String accountId) {
        throw new UnsupportedOperationException("No bank feed provider configured");
    }

    @Override
    public Page<Transaction> listTransactions(final String accountId,
            final Instant from, final Instant to, final PageRequest pagination) {
        return Page.of(List.of());
    }

    @Override
    public Transaction getTransaction(final String accountId,
            final String transactionId) {
        throw new UnsupportedOperationException("No bank feed provider configured");
    }
}
```

- [ ] **Step 7: Run tests to verify they pass**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl bank-spi -Dtest=BankFeedPlatformServiceTest -f /Users/mdproctor/claude/casehub/connectors/pom.xml`
Expected: PASS (6 tests)

- [ ] **Step 8: Full build to verify module wiring**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn clean install -f /Users/mdproctor/claude/casehub/connectors/pom.xml`
Expected: BUILD SUCCESS

- [ ] **Step 9: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/connectors add bank-spi/ pom.xml
git -C /Users/mdproctor/claude/casehub/connectors commit -m "feat(bank): add BankFeedPlatform SPI with @SimulationEligible — Refs #94"
```

## Batch 2: EmailPlatform SPI + Documentation

### Task 3: email-spi module — SPI, models, service, no-op, tests

**Files:**
- Create: `email-spi/pom.xml`
- Create: `email-spi/src/main/java/io/casehub/connectors/email/spi/EmailPlatform.java`
- Create: `email-spi/src/main/java/io/casehub/connectors/email/spi/EmailPlatformService.java`
- Create: `email-spi/src/main/java/io/casehub/connectors/email/spi/EmailBeans.java`
- Create: `email-spi/src/main/java/io/casehub/connectors/email/spi/NoOpEmailPlatform.java`
- Create: `email-spi/src/main/java/io/casehub/connectors/email/model/Mailbox.java`
- Create: `email-spi/src/main/java/io/casehub/connectors/email/model/EmailSummary.java`
- Create: `email-spi/src/main/java/io/casehub/connectors/email/model/EmailMessage.java`
- Create: `email-spi/src/main/java/io/casehub/connectors/email/model/EmailAttachment.java`
- Modify: `pom.xml` (parent — add `<module>email-spi</module>`)
- Test: `email-spi/src/test/java/io/casehub/connectors/email/spi/EmailPlatformServiceTest.java`

**Interfaces:**
- Consumes: `Page<T>`, `PageRequest` from connectors-api (Task 1)
- Produces: `EmailPlatform` SPI interface, `EmailPlatformService.platform(id)`, all model records

Note: Service, beans, and no-op live in `io.casehub.connectors.email.spi` (not `io.casehub.connectors.email`) to avoid split-package with the existing `email` module's `EmailConnector` in `io.casehub.connectors.email`.

- [ ] **Step 1: Create email-spi/pom.xml and add to parent**

Create `email-spi/pom.xml`:

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

  <artifactId>casehub-connectors-email-spi</artifactId>
  <name>CaseHub Connectors — Email Platform SPI</name>
  <description>Email Platform SPI: EmailPlatform interface, model types,
and EmailPlatformService routing. Complements EmailConnector (outbound)
and EmailInboundConnector (push inbound) with query/read capabilities.</description>

  <dependencies>
    <dependency>
      <groupId>io.casehub</groupId>
      <artifactId>casehub-connectors-api</artifactId>
      <version>${project.version}</version>
    </dependency>
    <dependency>
      <groupId>io.casehub</groupId>
      <artifactId>casehub-platform-simulation-api</artifactId>
      <version>${project.version}</version>
    </dependency>
    <dependency>
      <groupId>io.quarkus</groupId>
      <artifactId>quarkus-arc</artifactId>
    </dependency>

    <!-- Testing -->
    <dependency>
      <groupId>io.quarkus</groupId>
      <artifactId>quarkus-junit</artifactId>
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
    </plugins>
  </build>

</project>
```

Add `<module>email-spi</module>` to the parent pom's `<modules>` section (after `bank-spi`).

- [ ] **Step 2: Create model records**

Create `email-spi/src/main/java/io/casehub/connectors/email/model/Mailbox.java`:

```java
package io.casehub.connectors.email.model;

public record Mailbox(String id, String name, int unreadCount) {}
```

Create `email-spi/src/main/java/io/casehub/connectors/email/model/EmailSummary.java`:

```java
package io.casehub.connectors.email.model;

import java.time.Instant;

public record EmailSummary(String id, String mailboxId,
                           String messageId,
                           String from, String subject,
                           Instant receivedAt, boolean read) {}
```

Create `email-spi/src/main/java/io/casehub/connectors/email/model/EmailMessage.java`:

```java
package io.casehub.connectors.email.model;

import java.time.Instant;
import java.util.List;

public record EmailMessage(String id, String mailboxId,
                           String messageId,
                           String from, List<String> to, List<String> cc,
                           String subject, String bodyText, String bodyHtml,
                           Instant receivedAt, boolean read,
                           List<EmailAttachment> attachments) {}
```

Create `email-spi/src/main/java/io/casehub/connectors/email/model/EmailAttachment.java`:

```java
package io.casehub.connectors.email.model;

public record EmailAttachment(String id, String filename,
                              String contentType, long size) {}
```

- [ ] **Step 3: Create EmailPlatform SPI interface**

Create `email-spi/src/main/java/io/casehub/connectors/email/spi/EmailPlatform.java`:

```java
package io.casehub.connectors.email.spi;

import java.time.Instant;
import java.util.List;

import io.casehub.connectors.Page;
import io.casehub.connectors.PageRequest;
import io.casehub.connectors.email.model.EmailMessage;
import io.casehub.connectors.email.model.EmailSummary;
import io.casehub.connectors.email.model.Mailbox;
import io.casehub.platform.simulation.SimulationEligible;

@SimulationEligible(name = "email-platform")
public interface EmailPlatform {

    String id();

    List<Mailbox> listMailboxes();

    Page<EmailSummary> listMessages(String mailboxId,
                                    Instant from, Instant to,
                                    PageRequest pagination);

    EmailMessage getMessage(String mailboxId, String messageId);

    byte[] getAttachmentContent(String mailboxId, String messageId,
                                String attachmentId);
}
```

- [ ] **Step 4: Write failing test for EmailPlatformService**

Create `email-spi/src/test/java/io/casehub/connectors/email/spi/EmailPlatformServiceTest.java`:

```java
package io.casehub.connectors.email.spi;

import java.time.Instant;
import java.util.List;

import org.junit.jupiter.api.Test;

import io.casehub.connectors.Page;
import io.casehub.connectors.PageRequest;
import io.casehub.connectors.email.model.EmailMessage;
import io.casehub.connectors.email.model.EmailSummary;
import io.casehub.connectors.email.model.Mailbox;

import static org.assertj.core.api.Assertions.assertThat;
import static org.assertj.core.api.Assertions.assertThatThrownBy;

class EmailPlatformServiceTest {

    static class StubPlatform implements EmailPlatform {
        private final String platformId;
        StubPlatform(String id) { this.platformId = id; }
        @Override public String id() { return platformId; }
        @Override public List<Mailbox> listMailboxes() { return List.of(); }
        @Override public Page<EmailSummary> listMessages(String m, Instant f, Instant t, PageRequest p) { return Page.of(List.of()); }
        @Override public EmailMessage getMessage(String m, String id) { return null; }
        @Override public byte[] getAttachmentContent(String m, String mid, String aid) { return new byte[0]; }
    }

    @Test
    void platform_knownId_returnsPlatform() {
        var service = new EmailPlatformService(List.of(new StubPlatform("imap")));
        assertThat(service.platform("imap").id()).isEqualTo("imap");
    }

    @Test
    void platform_unknownId_throws() {
        var service = new EmailPlatformService(List.of(new StubPlatform("imap")));
        assertThatThrownBy(() -> service.platform("graph"))
                .isInstanceOf(IllegalArgumentException.class)
                .hasMessageContaining("graph")
                .hasMessageContaining("imap");
    }

    @Test
    void supports_knownId_returnsTrue() {
        var service = new EmailPlatformService(List.of(new StubPlatform("imap")));
        assertThat(service.supports("imap")).isTrue();
    }

    @Test
    void supports_unknownId_returnsFalse() {
        var service = new EmailPlatformService(List.of(new StubPlatform("imap")));
        assertThat(service.supports("graph")).isFalse();
    }

    @Test
    void ids_returnsAllRegistered() {
        var service = new EmailPlatformService(
                List.of(new StubPlatform("imap"), new StubPlatform("graph")));
        assertThat(service.ids()).containsExactlyInAnyOrder("imap", "graph");
    }

    @Test
    void duplicateId_throwsAtConstruction() {
        assertThatThrownBy(() -> new EmailPlatformService(
                List.of(new StubPlatform("imap"), new StubPlatform("imap"))))
                .isInstanceOf(IllegalStateException.class)
                .hasMessageContaining("imap");
    }
}
```

- [ ] **Step 5: Run tests to verify they fail**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl email-spi -Dtest=EmailPlatformServiceTest -f /Users/mdproctor/claude/casehub/connectors/pom.xml`
Expected: FAIL — `EmailPlatformService` not found

- [ ] **Step 6: Implement EmailPlatformService, EmailBeans, NoOpEmailPlatform**

Create `email-spi/src/main/java/io/casehub/connectors/email/spi/EmailPlatformService.java`:

```java
package io.casehub.connectors.email.spi;

import java.util.List;
import java.util.Map;
import java.util.Set;
import java.util.function.Function;
import java.util.stream.Collectors;

public class EmailPlatformService {

    private final Map<String, EmailPlatform> registry;

    public EmailPlatformService(final List<EmailPlatform> platforms) {
        this.registry = platforms.stream()
                .collect(Collectors.toMap(
                        EmailPlatform::id,
                        Function.identity(),
                        (a, b) -> {
                            throw new IllegalStateException(
                                    "Duplicate email platform id: '" + a.id() + "'");
                        }));
    }

    public EmailPlatform platform(final String id) {
        final EmailPlatform platform = registry.get(id);
        if (platform == null) {
            throw new IllegalArgumentException(
                    "No email platform registered for id '" + id
                    + "'. Available: " + registry.keySet());
        }
        return platform;
    }

    public boolean supports(final String id) {
        return registry.containsKey(id);
    }

    public Set<String> ids() {
        return Set.copyOf(registry.keySet());
    }
}
```

Create `email-spi/src/main/java/io/casehub/connectors/email/spi/EmailBeans.java`:

```java
package io.casehub.connectors.email.spi;

import io.quarkus.arc.All;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.enterprise.inject.Produces;

import java.util.List;

@ApplicationScoped
public class EmailBeans {

    @Produces
    @ApplicationScoped
    public EmailPlatformService emailPlatformService(
            @All List<EmailPlatform> platforms) {
        return new EmailPlatformService(platforms);
    }
}
```

Create `email-spi/src/main/java/io/casehub/connectors/email/spi/NoOpEmailPlatform.java`:

```java
package io.casehub.connectors.email.spi;

import java.time.Instant;
import java.util.List;

import io.casehub.connectors.Page;
import io.casehub.connectors.PageRequest;
import io.casehub.connectors.email.model.EmailMessage;
import io.casehub.connectors.email.model.EmailSummary;
import io.casehub.connectors.email.model.Mailbox;
import io.quarkus.arc.DefaultBean;
import jakarta.enterprise.context.ApplicationScoped;

@DefaultBean
@ApplicationScoped
public class NoOpEmailPlatform implements EmailPlatform {

    @Override
    public String id() {
        return "none";
    }

    @Override
    public List<Mailbox> listMailboxes() {
        return List.of();
    }

    @Override
    public Page<EmailSummary> listMessages(final String mailboxId,
            final Instant from, final Instant to, final PageRequest pagination) {
        return Page.of(List.of());
    }

    @Override
    public EmailMessage getMessage(final String mailboxId,
            final String messageId) {
        throw new UnsupportedOperationException("No email provider configured");
    }

    @Override
    public byte[] getAttachmentContent(final String mailboxId,
            final String messageId, final String attachmentId) {
        throw new UnsupportedOperationException("No email provider configured");
    }
}
```

- [ ] **Step 7: Run tests to verify they pass**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl email-spi -Dtest=EmailPlatformServiceTest -f /Users/mdproctor/claude/casehub/connectors/pom.xml`
Expected: PASS (6 tests)

- [ ] **Step 8: Full build to verify all modules**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn clean install -f /Users/mdproctor/claude/casehub/connectors/pom.xml`
Expected: BUILD SUCCESS

- [ ] **Step 9: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/connectors add email-spi/ pom.xml
git -C /Users/mdproctor/claude/casehub/connectors commit -m "feat(email): add EmailPlatform SPI with @SimulationEligible — Refs #94"
```

### Task 4: Documentation updates

**Files:**
- Modify: `CLAUDE.md` (add bank-spi and email-spi to module table)
- Modify: `docs/guides/consumer-guide.md` (add bank-spi and email-spi sections)
- Modify: `ARC42STORIES.MD` (add bank-feed and email-platform layers)

**Interfaces:**
- Consumes: All types from Task 2 and Task 3
- Produces: Updated documentation

- [ ] **Step 1: Update CLAUDE.md module table**

Add two rows to the `## Modules` table in `CLAUDE.md`:

```
| `bank-spi` | BankFeedPlatform SPI |
| `email-spi` | EmailPlatform SPI |
```

Update the project description paragraph to mention `BankPlatform` and `EmailPlatform` SPIs.

- [ ] **Step 2: Update consumer guide**

Add a `## Bank Feed Platform` section and an `## Email Platform` section to `docs/guides/consumer-guide.md` covering:
- Module dependency coordinates
- Interface methods
- Model records
- Platform service usage
- Simulation integration (qualified names, test fixture example)

Follow the existing guide structure for ChatPlatform and CalendarPlatform sections.

- [ ] **Step 3: Update ARC42STORIES.MD**

Add entries for BankFeedPlatform and EmailPlatform in the building block view (§5) and module structure. Follow the existing pattern for CalendarPlatform and ChatPlatform entries.

- [ ] **Step 4: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/connectors add CLAUDE.md docs/guides/consumer-guide.md ARC42STORIES.MD
git -C /Users/mdproctor/claude/casehub/connectors commit -m "docs: add BankFeedPlatform and EmailPlatform to module docs — Refs #94"
```

## References

- [2026-09-20-bankfeed-email-platform-spis-design.md] — design spec
- [decisions.md] — 11 decisions including simulation framework integration
- [calendar-spi/CalendarPlatform.java] — flat SPI pattern template
- [calendar-spi/CalendarPlatformService.java] — registry pattern template
- [calendar-spi/CalendarBeans.java] — CDI producer pattern
- [calendar-spi/CalendarPlatformServiceTest.java] — test pattern template
- [calendar-spi/pom.xml] — module pom template
- [connectors-api/] — shared types package (Page, PageRequest destination)
- [platform/simulation-api/SimulationEligible.java] — @SimulationEligible annotation
- casehubio/connectors#94 — focal issue
- casehubio/platform#370 — YAML parity gaps (framework dependency)
- PP-20260609-e3a2bd — SPI id() naming protocol
