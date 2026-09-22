# TrueLayer BankPlatform Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #106 — First real provider — TrueLayer BankFeedPlatform implementation
**Issue group:** #106

**Goal:** Evolve BankFeedPlatform into a capability-based BankPlatform SPI and implement TrueLayer as the first real provider with AISP + PISP support.

**Architecture:** Rename `BankFeedPlatform` → `BankPlatform` with `AccountInformation` and `PaymentInitiation` capability sub-interfaces (ChatPlatform pattern). New `bank-truelayer` module provides `TrueLayerClient` (HttpHelper.CLIENT + quarkus-oidc-client), `TrueLayerConsentService` (PSD2 consent lifecycle), and `TrueLayerBankPlatform`. GraphQL module migrated to capability accessors.

**Tech Stack:** Java 21, Quarkus 3.32.2, quarkus-oidc-client, quarkus-rest, WireMock, HttpHelper.CLIENT

## Global Constraints

- All outbound HTTP calls use `HttpHelper.CLIENT` (PP-20260607-9794cb)
- Credentials at call time, not `@ConfigProperty` on shared clients (PP-20260609-0c3e24)
- Paginating methods return partial results + WARNING on mid-loop failure (PP-20260610-83747b)
- SPI identifier methods named `id()` (PP-20260609-e3a2bd)
- `BigDecimal` for monetary amounts — never floating point
- `@SimulationEligible` with `capabilities` attribute for recursive wrapper generation (platform#375)
- All cross-project artifacts are `0.2-SNAPSHOT`
- Build: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn clean install`

---

## Batch 1: BankPlatform SPI Evolution

### Task 1: Rename BankFeedPlatform → BankPlatform with capability sub-interfaces

**Files:**
- Rename: `BankFeedPlatform` → `BankPlatform` (use `ide_refactor_rename`)
- Create: `bank-spi/src/main/java/io/casehub/connectors/bank/spi/AccountInformation.java`
- Create: `bank-spi/src/main/java/io/casehub/connectors/bank/spi/PaymentInitiation.java`
- Modify: `bank-spi/src/main/java/io/casehub/connectors/bank/spi/BankPlatform.java` (after rename)
- Rename: `BankFeedPlatformService` → `BankPlatformService` (use `ide_refactor_rename`)
- Rename: `NoOpBankFeedPlatform` → `NoOpBankPlatform` (use `ide_refactor_rename`)
- Rename: `BankFeedPlatformServiceTest` → `BankPlatformServiceTest` (use `ide_refactor_rename`)
- Create: `bank-spi/src/main/java/io/casehub/connectors/bank/NoOpAccountInformation.java`
- Create: `bank-spi/src/main/java/io/casehub/connectors/bank/NoOpPaymentInitiation.java`
- Modify: `bank-spi/pom.xml` (update name/description)
- Test: `bank-spi/src/test/java/io/casehub/connectors/bank/BankPlatformServiceTest.java` (after rename)

**Interfaces:**
- Produces: `BankPlatform` interface with `id()`, `accountInformation(String userId)`, `paymentInitiation(String userId)`, `supports(Class<?>)`
- Produces: `AccountInformation` interface with `listAccounts()`, `balance(String)`, `listTransactions(...)`, `getTransaction(...)`
- Produces: `PaymentInitiation` interface with `initiatePayment(PaymentRequest)`, `paymentStatus(String)`
- Produces: `BankPlatformService` with `platform(String)`, `supports(String)`, `ids()`

- [ ] **Step 1: Use `ide_refactor_rename` to rename `BankFeedPlatform` → `BankPlatform`**

Use `ide_refactor_rename` on `BankFeedPlatform.java`. This updates all references across the project (imports in `NoOpBankFeedPlatform`, `BankFeedPlatformService`, `BankBeans`, `ConnectorBankFeedApi`, and the test).

- [ ] **Step 2: Use `ide_refactor_rename` to rename `BankFeedPlatformService` → `BankPlatformService`**

- [ ] **Step 3: Use `ide_refactor_rename` to rename `NoOpBankFeedPlatform` → `NoOpBankPlatform`**

- [ ] **Step 4: Use `ide_refactor_rename` to rename `BankFeedPlatformServiceTest` → `BankPlatformServiceTest`**

- [ ] **Step 5: Create `AccountInformation` interface**

Create `bank-spi/src/main/java/io/casehub/connectors/bank/spi/AccountInformation.java`:

```java
package io.casehub.connectors.bank.spi;

import java.time.Instant;
import java.util.List;

import io.casehub.connectors.Page;
import io.casehub.connectors.PageRequest;
import io.casehub.connectors.bank.model.AccountBalance;
import io.casehub.connectors.bank.model.AccountInfo;
import io.casehub.connectors.bank.model.Transaction;

public interface AccountInformation {

    List<AccountInfo> listAccounts();

    AccountBalance balance(String accountId);

    Page<Transaction> listTransactions(String accountId,
                                       Instant from, Instant to,
                                       PageRequest pagination);

    Transaction getTransaction(String accountId, String transactionId);
}
```

- [ ] **Step 6: Create payment model records**

Create `bank-spi/src/main/java/io/casehub/connectors/bank/model/PaymentDestination.java`:

```java
package io.casehub.connectors.bank.model;

public sealed interface PaymentDestination {
    record UkAccount(String sortCode, String accountNumber)
            implements PaymentDestination {}
    record IbanAccount(String iban, String bic)
            implements PaymentDestination {}
}
```

Create `bank-spi/src/main/java/io/casehub/connectors/bank/model/PaymentRequest.java`:

```java
package io.casehub.connectors.bank.model;

import java.math.BigDecimal;

public record PaymentRequest(String idempotencyKey,
                             BigDecimal amount, String currency,
                             String beneficiaryName,
                             PaymentDestination destination,
                             String reference) {}
```

Create `bank-spi/src/main/java/io/casehub/connectors/bank/model/InitiatedPayment.java`:

```java
package io.casehub.connectors.bank.model;

public record InitiatedPayment(String paymentId,
                                String hostedPaymentPageLink,
                                PaymentStatus status) {}
```

Create `bank-spi/src/main/java/io/casehub/connectors/bank/model/PaymentStatus.java`:

```java
package io.casehub.connectors.bank.model;

public enum PaymentStatus {
    AUTHORIZATION_REQUIRED, AUTHORIZING, AUTHORIZED,
    EXECUTED, SETTLED, FAILED
}
```

- [ ] **Step 7: Create `PaymentInitiation` interface**

Create `bank-spi/src/main/java/io/casehub/connectors/bank/spi/PaymentInitiation.java`:

```java
package io.casehub.connectors.bank.spi;

import io.casehub.connectors.bank.model.InitiatedPayment;
import io.casehub.connectors.bank.model.PaymentRequest;
import io.casehub.connectors.bank.model.PaymentStatus;

public interface PaymentInitiation {

    InitiatedPayment initiatePayment(PaymentRequest payment);

    PaymentStatus paymentStatus(String paymentId);
}
```

- [ ] **Step 8: Rewrite `BankPlatform` as capability-based interface**

Use `ide_replace_member` or Edit to replace the interface body of `BankPlatform.java`:

```java
package io.casehub.connectors.bank.spi;

import io.casehub.platform.simulation.SimulationEligible;

@SimulationEligible(name = "bank-platform",
    capabilities = {"accountInformation", "paymentInitiation"})
public interface BankPlatform {

    String id();

    AccountInformation accountInformation(String userId);

    PaymentInitiation paymentInitiation(String userId);

    boolean supports(Class<?> capability);
}
```

- [ ] **Step 9: Create `NoOpAccountInformation`**

Create `bank-spi/src/main/java/io/casehub/connectors/bank/NoOpAccountInformation.java`:

```java
package io.casehub.connectors.bank;

import java.time.Instant;
import java.util.List;

import io.casehub.connectors.Page;
import io.casehub.connectors.PageRequest;
import io.casehub.connectors.bank.model.AccountBalance;
import io.casehub.connectors.bank.model.AccountInfo;
import io.casehub.connectors.bank.model.Transaction;
import io.casehub.connectors.bank.spi.AccountInformation;

public class NoOpAccountInformation implements AccountInformation {

    public static final NoOpAccountInformation INSTANCE = new NoOpAccountInformation();

    @Override
    public List<AccountInfo> listAccounts() {
        return List.of();
    }

    @Override
    public AccountBalance balance(String accountId) {
        throw new UnsupportedOperationException("No bank platform configured");
    }

    @Override
    public Page<Transaction> listTransactions(String accountId,
            Instant from, Instant to, PageRequest pagination) {
        return Page.of(List.of());
    }

    @Override
    public Transaction getTransaction(String accountId, String transactionId) {
        throw new UnsupportedOperationException("No bank platform configured");
    }
}
```

- [ ] **Step 10: Create `NoOpPaymentInitiation`**

Create `bank-spi/src/main/java/io/casehub/connectors/bank/NoOpPaymentInitiation.java`:

```java
package io.casehub.connectors.bank;

import io.casehub.connectors.bank.model.InitiatedPayment;
import io.casehub.connectors.bank.model.PaymentRequest;
import io.casehub.connectors.bank.model.PaymentStatus;
import io.casehub.connectors.bank.spi.PaymentInitiation;

public class NoOpPaymentInitiation implements PaymentInitiation {

    public static final NoOpPaymentInitiation INSTANCE = new NoOpPaymentInitiation();

    @Override
    public InitiatedPayment initiatePayment(PaymentRequest payment) {
        throw new UnsupportedOperationException("No bank platform configured");
    }

    @Override
    public PaymentStatus paymentStatus(String paymentId) {
        throw new UnsupportedOperationException("No bank platform configured");
    }
}
```

- [ ] **Step 11: Rewrite `NoOpBankPlatform` for capability pattern**

Replace the body of `NoOpBankPlatform.java`:

```java
package io.casehub.connectors.bank;

import io.casehub.connectors.bank.spi.AccountInformation;
import io.casehub.connectors.bank.spi.BankPlatform;
import io.casehub.connectors.bank.spi.PaymentInitiation;
import io.quarkus.arc.DefaultBean;
import jakarta.enterprise.context.ApplicationScoped;

@DefaultBean
@ApplicationScoped
public class NoOpBankPlatform implements BankPlatform {

    @Override
    public String id() {
        return "none";
    }

    @Override
    public AccountInformation accountInformation(String userId) {
        return NoOpAccountInformation.INSTANCE;
    }

    @Override
    public PaymentInitiation paymentInitiation(String userId) {
        return NoOpPaymentInitiation.INSTANCE;
    }

    @Override
    public boolean supports(Class<?> capability) {
        return false;
    }
}
```

- [ ] **Step 12: Update `BankBeans` and `BankPlatformService`**

`BankBeans.java` — update the produce method name and error message in `BankPlatformService`:

```java
package io.casehub.connectors.bank;

import io.casehub.connectors.bank.spi.BankPlatform;
import io.quarkus.arc.All;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.enterprise.inject.Produces;

import java.util.List;

@ApplicationScoped
public class BankBeans {

    @Produces
    @ApplicationScoped
    public BankPlatformService bankPlatformService(
            @All List<BankPlatform> platforms) {
        return new BankPlatformService(platforms);
    }
}
```

`BankPlatformService.java` — update types and error messages:

```java
package io.casehub.connectors.bank;

import java.util.List;
import java.util.Map;
import java.util.Set;
import java.util.function.Function;
import java.util.stream.Collectors;

import io.casehub.connectors.bank.spi.BankPlatform;

public class BankPlatformService {

    private final Map<String, BankPlatform> registry;

    public BankPlatformService(final List<BankPlatform> platforms) {
        this.registry = platforms.stream()
                .collect(Collectors.toMap(
                        BankPlatform::id,
                        Function.identity(),
                        (a, b) -> {
                            throw new IllegalStateException(
                                    "Duplicate bank platform id: '" + a.id() + "'");
                        }));
    }

    public BankPlatform platform(final String id) {
        final BankPlatform platform = registry.get(id);
        if (platform == null) {
            throw new IllegalArgumentException(
                    "No bank platform registered for id '" + id
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

- [ ] **Step 13: Update `BankPlatformServiceTest` for capability pattern**

```java
package io.casehub.connectors.bank;

import java.util.List;

import org.junit.jupiter.api.Test;

import io.casehub.connectors.bank.spi.AccountInformation;
import io.casehub.connectors.bank.spi.BankPlatform;
import io.casehub.connectors.bank.spi.PaymentInitiation;

import static org.assertj.core.api.Assertions.assertThat;
import static org.assertj.core.api.Assertions.assertThatThrownBy;

class BankPlatformServiceTest {

    static class StubPlatform implements BankPlatform {
        private final String platformId;
        StubPlatform(String id) { this.platformId = id; }
        @Override public String id() { return platformId; }
        @Override public AccountInformation accountInformation(String userId) {
            return NoOpAccountInformation.INSTANCE;
        }
        @Override public PaymentInitiation paymentInitiation(String userId) {
            return NoOpPaymentInitiation.INSTANCE;
        }
        @Override public boolean supports(Class<?> capability) {
            return capability == AccountInformation.class;
        }
    }

    @Test
    void platform_knownId_returnsPlatform() {
        var service = new BankPlatformService(List.of(new StubPlatform("truelayer")));
        assertThat(service.platform("truelayer").id()).isEqualTo("truelayer");
    }

    @Test
    void platform_unknownId_throws() {
        var service = new BankPlatformService(List.of(new StubPlatform("truelayer")));
        assertThatThrownBy(() -> service.platform("plaid"))
                .isInstanceOf(IllegalArgumentException.class)
                .hasMessageContaining("plaid")
                .hasMessageContaining("truelayer");
    }

    @Test
    void supports_knownId_returnsTrue() {
        var service = new BankPlatformService(List.of(new StubPlatform("truelayer")));
        assertThat(service.supports("truelayer")).isTrue();
    }

    @Test
    void supports_unknownId_returnsFalse() {
        var service = new BankPlatformService(List.of(new StubPlatform("truelayer")));
        assertThat(service.supports("plaid")).isFalse();
    }

    @Test
    void ids_returnsAllRegistered() {
        var service = new BankPlatformService(
                List.of(new StubPlatform("truelayer"), new StubPlatform("plaid")));
        assertThat(service.ids()).containsExactlyInAnyOrder("truelayer", "plaid");
    }

    @Test
    void duplicateId_throwsAtConstruction() {
        assertThatThrownBy(() -> new BankPlatformService(
                List.of(new StubPlatform("truelayer"), new StubPlatform("truelayer"))))
                .isInstanceOf(IllegalStateException.class)
                .hasMessageContaining("truelayer");
    }
}
```

- [ ] **Step 14: Update `bank-spi/pom.xml` name and description**

Change `<name>` to `CaseHub Connectors — Bank SPI` and `<description>` to `Bank Platform SPI: BankPlatform interface with AccountInformation and PaymentInitiation capabilities, model types, and BankPlatformService routing.`

- [ ] **Step 15: Build and verify**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn -pl bank-spi -am clean install`
Expected: BUILD SUCCESS

- [ ] **Step 16: Commit**

```bash
git add bank-spi/ graphql/
git commit -m "feat(#106): rename BankFeedPlatform → BankPlatform with capability sub-interfaces

Refs #106"
```

### Task 2: Migrate GraphQL module to capability accessors

**Files:**
- Rename: `ConnectorBankFeedApi` → `ConnectorBankApi` (use `ide_refactor_rename`)
- Modify: `graphql/src/main/java/io/casehub/connectors/graphql/ConnectorBankApi.java` (after rename)
- Test: build the graphql module

**Interfaces:**
- Consumes: `BankPlatform.accountInformation(String userId)` → `AccountInformation`
- Consumes: `BankPlatformService.platform(String)` → `BankPlatform`

- [ ] **Step 1: Use `ide_refactor_rename` to rename `ConnectorBankFeedApi` → `ConnectorBankApi`**

- [ ] **Step 2: Rewrite `ConnectorBankApi` for capability accessors**

```java
package io.casehub.connectors.graphql;

import io.casehub.connectors.bank.BankPlatformService;
import io.casehub.connectors.bank.model.AccountBalance;
import io.casehub.connectors.bank.model.AccountInfo;
import io.casehub.connectors.bank.model.InitiatedPayment;
import io.casehub.connectors.bank.model.PaymentRequest;
import io.casehub.connectors.bank.model.PaymentStatus;
import io.casehub.connectors.Page;
import io.casehub.connectors.PageRequest;
import io.casehub.connectors.bank.model.Transaction;
import io.casehub.connectors.bank.spi.BankPlatform;
import io.casehub.platform.api.mcp.McpDomain;
import io.casehub.platform.api.mcp.PathParam;
import io.casehub.platform.api.mcp.PlatformMutation;
import io.casehub.platform.api.mcp.PlatformQuery;
import io.casehub.platform.api.mcp.RestPath;
import io.quarkus.security.identity.SecurityIdentity;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.inject.Inject;
import jakarta.ws.rs.QueryParam;

import java.time.Instant;
import java.util.List;

@McpDomain(value = "connectors/bank", basePath = "/api/connectors/bank")
@ApplicationScoped
public class ConnectorBankApi {

    @Inject BankPlatformService bankService;
    @Inject SecurityIdentity identity;

    private String userId() {
        return identity.getPrincipal().getName();
    }

    @PlatformQuery("List bank accounts on a platform")
    @RestPath("/accounts")
    public List<AccountInfo> listAccounts(@QueryParam("platform") String platform) {
        BankPlatform p = bankService.platform(platform);
        return p.accountInformation(userId()).listAccounts();
    }

    @PlatformQuery("Get current balance for a bank account")
    @RestPath("/accounts/{accountId}/balance")
    public AccountBalance balance(
            @QueryParam("platform") String platform,
            @PathParam String accountId) {
        BankPlatform p = bankService.platform(platform);
        return p.accountInformation(userId()).balance(accountId);
    }

    @PlatformQuery("List transactions for a bank account")
    @RestPath("/accounts/{accountId}/transactions")
    public Page<Transaction> listTransactions(
            @QueryParam("platform") String platform,
            @PathParam String accountId,
            @QueryParam("from") Instant from,
            @QueryParam("to") Instant to,
            @QueryParam("cursor") String cursor,
            @QueryParam("pageSize") Integer pageSize) {
        BankPlatform p = bankService.platform(platform);
        Instant effectiveFrom = from != null ? from : Instant.now().minusSeconds(2592000);
        Instant effectiveTo = to != null ? to : Instant.now();
        int size = pageSize != null ? pageSize : 50;
        return p.accountInformation(userId()).listTransactions(
                accountId, effectiveFrom, effectiveTo, new PageRequest(cursor, size));
    }

    @PlatformQuery("Get a single transaction by ID")
    @RestPath("/accounts/{accountId}/transactions/{transactionId}")
    public Transaction getTransaction(
            @QueryParam("platform") String platform,
            @PathParam String accountId,
            @PathParam String transactionId) {
        BankPlatform p = bankService.platform(platform);
        return p.accountInformation(userId()).getTransaction(accountId, transactionId);
    }

    @PlatformMutation("Initiate a payment")
    @RestPath("/payments")
    public InitiatedPayment initiatePayment(
            @QueryParam("platform") String platform,
            PaymentRequest payment) {
        BankPlatform p = bankService.platform(platform);
        return p.paymentInitiation(userId()).initiatePayment(payment);
    }

    @PlatformQuery("Get payment status")
    @RestPath("/payments/{paymentId}")
    public PaymentStatus paymentStatus(
            @QueryParam("platform") String platform,
            @PathParam String paymentId) {
        BankPlatform p = bankService.platform(platform);
        return p.paymentInitiation(userId()).paymentStatus(paymentId);
    }
}
```

- [ ] **Step 3: Build and verify**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn -pl graphql -am clean install`
Expected: BUILD SUCCESS

- [ ] **Step 4: Commit**

```bash
git add graphql/
git commit -m "feat(#106): migrate GraphQL bank API to capability accessors

Refs #106"
```

## Batch 2: TrueLayer Module Foundation — Client and Consent

### Task 3: Create bank-truelayer module with TrueLayerClient

**Files:**
- Create: `bank-truelayer/pom.xml`
- Create: `bank-truelayer/src/main/java/io/casehub/connectors/bank/truelayer/TrueLayerClient.java`
- Create: `bank-truelayer/src/main/java/io/casehub/connectors/bank/truelayer/dto/TrueLayerAccount.java`
- Create: `bank-truelayer/src/main/java/io/casehub/connectors/bank/truelayer/dto/TrueLayerBalance.java`
- Create: `bank-truelayer/src/main/java/io/casehub/connectors/bank/truelayer/dto/TrueLayerTransaction.java`
- Create: `bank-truelayer/src/main/java/io/casehub/connectors/bank/truelayer/dto/TrueLayerTransactionPage.java`
- Create: `bank-truelayer/src/main/java/io/casehub/connectors/bank/truelayer/dto/TrueLayerPaymentRequest.java`
- Create: `bank-truelayer/src/main/java/io/casehub/connectors/bank/truelayer/dto/TrueLayerPaymentResult.java`
- Create: `bank-truelayer/src/main/java/io/casehub/connectors/bank/truelayer/dto/TrueLayerPaymentStatus.java`
- Create: `bank-truelayer/src/main/java/io/casehub/connectors/bank/truelayer/ConsentExpiredException.java`
- Create: `bank-truelayer/src/main/java/io/casehub/connectors/bank/truelayer/RateLimitedException.java`
- Create: `bank-truelayer/src/main/java/io/casehub/connectors/bank/truelayer/TrueLayerApiException.java`
- Modify: `pom.xml` (parent — add `<module>bank-truelayer</module>`)
- Test: `bank-truelayer/src/test/java/io/casehub/connectors/bank/truelayer/TrueLayerClientTest.java`

**Interfaces:**
- Consumes: `HttpHelper.CLIENT` from `connectors-api`
- Consumes: `OidcClient` from `quarkus-oidc-client`
- Produces: `TrueLayerClient` with data API methods (listAccounts, balance, listTransactions, getTransaction) and payments API methods (createPayment, paymentStatus)

- [ ] **Step 1: Create `bank-truelayer/pom.xml`**

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

  <artifactId>casehub-connectors-bank-truelayer</artifactId>
  <name>CaseHub Connectors — TrueLayer Bank Provider</name>
  <description>TrueLayer provider for the BankPlatform SPI.
AISP (account information) and PISP (payment initiation) with PSD2 consent management.</description>

  <dependencies>
    <dependency>
      <groupId>io.casehub</groupId>
      <artifactId>casehub-connectors-bank-spi</artifactId>
      <version>${project.version}</version>
    </dependency>
    <dependency>
      <groupId>io.casehub</groupId>
      <artifactId>casehub-connectors-api</artifactId>
      <version>${project.version}</version>
    </dependency>
    <dependency>
      <groupId>io.quarkus</groupId>
      <artifactId>quarkus-oidc-client</artifactId>
    </dependency>
    <dependency>
      <groupId>io.quarkus</groupId>
      <artifactId>quarkus-rest</artifactId>
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
    <dependency>
      <groupId>org.wiremock</groupId>
      <artifactId>wiremock-standalone</artifactId>
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

- [ ] **Step 2: Add `bank-truelayer` to parent pom modules**

Add `<module>bank-truelayer</module>` after `<module>bank-spi</module>` in `pom.xml`.

- [ ] **Step 3: Create TrueLayer DTO records**

Create the following records in `bank-truelayer/src/main/java/io/casehub/connectors/bank/truelayer/dto/`:

`TrueLayerAccount.java`:
```java
package io.casehub.connectors.bank.truelayer.dto;

public record TrueLayerAccount(String accountId, String displayName,
                                String accountType, String currency) {}
```

`TrueLayerBalance.java`:
```java
package io.casehub.connectors.bank.truelayer.dto;

import java.math.BigDecimal;
import java.time.Instant;

public record TrueLayerBalance(String accountId,
                                BigDecimal available, BigDecimal current,
                                String currency, Instant updateTimestamp) {}
```

`TrueLayerTransaction.java`:
```java
package io.casehub.connectors.bank.truelayer.dto;

import java.math.BigDecimal;
import java.time.LocalDate;

public record TrueLayerTransaction(String transactionId, String accountId,
                                    BigDecimal amount, String currency,
                                    String transactionType,
                                    String description, String merchantName,
                                    String transactionCategory,
                                    LocalDate timestamp, String status) {}
```

`TrueLayerTransactionPage.java`:
```java
package io.casehub.connectors.bank.truelayer.dto;

import java.util.List;

public record TrueLayerTransactionPage(List<TrueLayerTransaction> results,
                                        String nextCursor, boolean hasMore) {}
```

`TrueLayerPaymentRequest.java`:
```java
package io.casehub.connectors.bank.truelayer.dto;

import java.math.BigDecimal;

public record TrueLayerPaymentRequest(BigDecimal amountInMinor, String currency,
                                       String beneficiaryName,
                                       String sortCode, String accountNumber,
                                       String reference) {}
```

`TrueLayerPaymentResult.java`:
```java
package io.casehub.connectors.bank.truelayer.dto;

public record TrueLayerPaymentResult(String id, String resourceToken,
                                      String hostedPaymentPageLink,
                                      String status) {}
```

`TrueLayerPaymentStatus.java`:
```java
package io.casehub.connectors.bank.truelayer.dto;

public record TrueLayerPaymentStatus(String id, String status,
                                      String failureReason) {}
```

- [ ] **Step 4: Create exception types**

`ConsentExpiredException.java`:
```java
package io.casehub.connectors.bank.truelayer;

public class ConsentExpiredException extends RuntimeException {
    public ConsentExpiredException(String userId) {
        super("Consent expired or not found for user '" + userId
              + "' — re-consent required via browser redirect");
    }
}
```

`RateLimitedException.java`:
```java
package io.casehub.connectors.bank.truelayer;

public class RateLimitedException extends RuntimeException {
    private final int retryAfterSeconds;
    public RateLimitedException(int retryAfterSeconds) {
        super("TrueLayer rate limit exceeded — retry after " + retryAfterSeconds + "s");
        this.retryAfterSeconds = retryAfterSeconds;
    }
    public int retryAfterSeconds() { return retryAfterSeconds; }
}
```

`TrueLayerApiException.java`:
```java
package io.casehub.connectors.bank.truelayer;

public class TrueLayerApiException extends RuntimeException {
    private final int statusCode;
    public TrueLayerApiException(int statusCode, String message) {
        super("TrueLayer API error (HTTP " + statusCode + "): " + message);
        this.statusCode = statusCode;
    }
    public int statusCode() { return statusCode; }
}
```

- [ ] **Step 5: Write failing test for TrueLayerClient.listAccounts**

Create `bank-truelayer/src/test/java/io/casehub/connectors/bank/truelayer/TrueLayerClientTest.java` with WireMock:

```java
package io.casehub.connectors.bank.truelayer;

import com.github.tomakehurst.wiremock.WireMockServer;
import org.junit.jupiter.api.AfterAll;
import org.junit.jupiter.api.BeforeAll;
import org.junit.jupiter.api.Test;

import static com.github.tomakehurst.wiremock.client.WireMock.*;
import static org.assertj.core.api.Assertions.assertThat;

class TrueLayerClientTest {

    static WireMockServer wireMock;
    static TrueLayerClient client;

    @BeforeAll
    static void setUp() {
        wireMock = new WireMockServer(0);
        wireMock.start();
        client = new TrueLayerClient(null, "http://localhost:" + wireMock.port());
    }

    @AfterAll
    static void tearDown() {
        wireMock.stop();
    }

    @Test
    void listAccounts_returnsAccounts() {
        wireMock.stubFor(get(urlPathEqualTo("/data/v1/accounts"))
                .willReturn(okJson("""
                    {"results": [
                        {"account_id": "acc-001", "display_name": "Current",
                         "account_type": "TRANSACTION", "currency": "GBP"}
                    ]}
                    """)));

        var accounts = client.listAccounts("user-token-123");
        assertThat(accounts).hasSize(1);
        assertThat(accounts.get(0).accountId()).isEqualTo("acc-001");
    }
}
```

- [ ] **Step 6: Run test — verify it fails**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn -pl bank-truelayer test -Dtest=TrueLayerClientTest#listAccounts_returnsAccounts`
Expected: FAIL (TrueLayerClient doesn't exist yet)

- [ ] **Step 7: Implement TrueLayerClient**

Create `bank-truelayer/src/main/java/io/casehub/connectors/bank/truelayer/TrueLayerClient.java`:

```java
package io.casehub.connectors.bank.truelayer;

import io.casehub.connectors.bank.truelayer.dto.*;
import io.casehub.connectors.http.HttpHelper;
import io.quarkus.oidc.client.OidcClient;
import org.jboss.logging.Logger;

import java.net.URI;
import java.net.http.HttpRequest;
import java.net.http.HttpResponse;
import java.util.ArrayList;
import java.util.Collections;
import java.util.List;
import java.util.NoSuchElementException;

public class TrueLayerClient {

    private static final Logger LOG = Logger.getLogger(TrueLayerClient.class);
    static final int MAX_PAGES = 50;

    private final OidcClient oidcClient;
    private final String baseUrl;

    public TrueLayerClient(OidcClient oidcClient, String baseUrl) {
        this.oidcClient = oidcClient;
        this.baseUrl = baseUrl;
    }

    public List<TrueLayerAccount> listAccounts(String userToken) {
        String json = get("/data/v1/accounts", userToken);
        return parseAccountList(json);
    }

    public TrueLayerBalance balance(String userToken, String accountId) {
        String json = get("/data/v1/accounts/" + accountId + "/balance", userToken);
        return parseBalance(json, accountId);
    }

    public TrueLayerTransactionPage listTransactions(String userToken,
            String accountId, String from, String to, String cursor) {
        String path = "/data/v1/accounts/" + accountId + "/transactions"
                + "?from=" + from + "&to=" + to;
        if (cursor != null) path += "&cursor=" + cursor;
        String json = get(path, userToken);
        return parseTransactionPage(json);
    }

    public TrueLayerTransaction getTransaction(String userToken,
            String accountId, String transactionId) {
        var page = listTransactions(userToken, accountId, null, null, null);
        return page.results().stream()
                .filter(t -> transactionId.equals(t.transactionId()))
                .findFirst()
                .orElseThrow(() -> new NoSuchElementException(
                        "Transaction '" + transactionId + "' not found"));
    }

    public TrueLayerPaymentResult createPayment(TrueLayerPaymentRequest request) {
        String clientToken = getClientToken();
        String json = post("/payments", clientToken, serializePaymentRequest(request));
        return parsePaymentResult(json);
    }

    public TrueLayerPaymentStatus paymentStatus(String paymentId) {
        String clientToken = getClientToken();
        String json = get("/payments/" + paymentId, clientToken);
        return parsePaymentStatus(json);
    }

    private String get(String path, String token) {
        try {
            HttpRequest request = HttpRequest.newBuilder()
                    .uri(URI.create(baseUrl + path))
                    .header("Authorization", "Bearer " + token)
                    .GET()
                    .build();
            HttpResponse<String> response = HttpHelper.CLIENT.send(
                    request, HttpResponse.BodyHandlers.ofString());
            return handleResponse(response);
        } catch (ConsentExpiredException | RateLimitedException | TrueLayerApiException e) {
            throw e;
        } catch (Exception e) {
            throw new TrueLayerApiException(0, e.getMessage());
        }
    }

    private String post(String path, String token, String body) {
        try {
            HttpRequest request = HttpRequest.newBuilder()
                    .uri(URI.create(baseUrl + path))
                    .header("Authorization", "Bearer " + token)
                    .header("Content-Type", "application/json")
                    .POST(HttpRequest.BodyPublishers.ofString(body))
                    .build();
            HttpResponse<String> response = HttpHelper.CLIENT.send(
                    request, HttpResponse.BodyHandlers.ofString());
            return handleResponse(response);
        } catch (ConsentExpiredException | RateLimitedException | TrueLayerApiException e) {
            throw e;
        } catch (Exception e) {
            throw new TrueLayerApiException(0, e.getMessage());
        }
    }

    private String handleResponse(HttpResponse<String> response) {
        int status = response.statusCode();
        if (status >= 200 && status < 300) return response.body();
        if (status == 401) throw new ConsentExpiredException("unknown");
        if (status == 404) throw new NoSuchElementException(response.body());
        if (status == 429) {
            int retryAfter = 60;
            String header = response.headers().firstValue("Retry-After").orElse(null);
            if (header != null) {
                try { retryAfter = Integer.parseInt(header); } catch (NumberFormatException ignored) {}
            }
            throw new RateLimitedException(retryAfter);
        }
        throw new TrueLayerApiException(status, response.body());
    }

    private String getClientToken() {
        if (oidcClient == null) throw new IllegalStateException("OidcClient not configured");
        return oidcClient.getTokens().await().indefinitely().getAccessToken();
    }

    // JSON parsing methods — minimal hand-rolled parsing using string operations
    // to avoid adding a JSON library dependency. Production would use Jackson/Jsonb.
    // For now, these parse the specific TrueLayer response formats.

    private List<TrueLayerAccount> parseAccountList(String json) {
        // Implementation: parse {"results": [...]} JSON
        // Delegate to a JsonParser utility or use jakarta.json
        return List.of(); // placeholder — implement with actual JSON parsing
    }

    private TrueLayerBalance parseBalance(String json, String accountId) {
        return null; // placeholder
    }

    private TrueLayerTransactionPage parseTransactionPage(String json) {
        return new TrueLayerTransactionPage(List.of(), null, false); // placeholder
    }

    private TrueLayerPaymentResult parsePaymentResult(String json) {
        return null; // placeholder
    }

    private TrueLayerPaymentStatus parsePaymentStatus(String json) {
        return null; // placeholder
    }

    private String serializePaymentRequest(TrueLayerPaymentRequest request) {
        return "{}"; // placeholder
    }
}
```

**Note to implementer:** The JSON parsing placeholders must be completed with actual parsing using `jakarta.json` (already available via Quarkus). Each parse method extracts fields from TrueLayer's JSON response format. Follow the pattern in `SlackBotClient` which does similar hand-parsing of Slack API JSON responses.

- [ ] **Step 8: Run test — verify it passes**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn -pl bank-truelayer test -Dtest=TrueLayerClientTest#listAccounts_returnsAccounts`
Expected: PASS

- [ ] **Step 9: Add more TrueLayerClient tests**

Add tests for: error mapping (401, 404, 429, 5xx), pagination with partial failure, balance retrieval, payment creation, payment status.

- [ ] **Step 10: Implement JSON parsing to pass all tests**

Complete the `parse*` and `serialize*` methods in `TrueLayerClient` using `jakarta.json`.

- [ ] **Step 11: Build and verify**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn -pl bank-truelayer -am clean install`
Expected: BUILD SUCCESS

- [ ] **Step 12: Commit**

```bash
git add bank-truelayer/ pom.xml
git commit -m "feat(#106): add bank-truelayer module with TrueLayerClient

Refs #106"
```

### Task 4: TrueLayerConsentService with state-based CSRF protection

**Files:**
- Create: `bank-truelayer/src/main/java/io/casehub/connectors/bank/truelayer/TrueLayerConsentService.java`
- Create: `bank-truelayer/src/main/java/io/casehub/connectors/bank/truelayer/StoredConsent.java`
- Create: `bank-truelayer/src/main/java/io/casehub/connectors/bank/truelayer/AuthLink.java`
- Create: `bank-truelayer/src/main/java/io/casehub/connectors/bank/truelayer/ConsentInfo.java`
- Create: `bank-truelayer/src/main/java/io/casehub/connectors/bank/truelayer/ConsentScope.java`
- Create: `bank-truelayer/src/main/java/io/casehub/connectors/bank/truelayer/ConsentStatus.java`
- Create: `bank-truelayer/src/main/java/io/casehub/connectors/bank/truelayer/InvalidStateException.java`
- Test: `bank-truelayer/src/test/java/io/casehub/connectors/bank/truelayer/TrueLayerConsentServiceTest.java`

**Interfaces:**
- Consumes: `TrueLayerClient` (for token refresh via TrueLayer token endpoint)
- Produces: `TrueLayerConsentService` with `generateAuthLink(userId, scopes, redirectUri)`, `exchangeCode(code, state)`, `consentStatus(userId)`, `revokeConsent(userId)`, `getUserToken(userId)`

- [ ] **Step 1: Create consent model records**

`ConsentScope.java`:
```java
package io.casehub.connectors.bank.truelayer;

public enum ConsentScope { ACCOUNTS, BALANCE, TRANSACTIONS, PAYMENTS }
```

`ConsentStatus.java`:
```java
package io.casehub.connectors.bank.truelayer;

public enum ConsentStatus { ACTIVE, EXPIRED, REVOKED }
```

`AuthLink.java`:
```java
package io.casehub.connectors.bank.truelayer;

public record AuthLink(String authorizationUrl) {}
```

`ConsentInfo.java`:
```java
package io.casehub.connectors.bank.truelayer;

import java.time.Instant;
import java.util.List;

public record ConsentInfo(String userId, ConsentStatus status,
                          Instant grantedAt, Instant expiresAt,
                          List<ConsentScope> scopes) {}
```

`StoredConsent.java`:
```java
package io.casehub.connectors.bank.truelayer;

import java.time.Instant;
import java.util.List;

public record StoredConsent(String accessToken, String refreshToken,
                            Instant accessTokenExpiry, Instant consentExpiry,
                            List<ConsentScope> scopes, Instant grantedAt) {}
```

`InvalidStateException.java`:
```java
package io.casehub.connectors.bank.truelayer;

public class InvalidStateException extends RuntimeException {
    public InvalidStateException(String message) { super(message); }
}
```

- [ ] **Step 2: Write failing test for consent flow**

Create `TrueLayerConsentServiceTest.java`:

```java
package io.casehub.connectors.bank.truelayer;

import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

import java.util.List;

import static org.assertj.core.api.Assertions.assertThat;
import static org.assertj.core.api.Assertions.assertThatThrownBy;

class TrueLayerConsentServiceTest {

    TrueLayerConsentService service;

    @BeforeEach
    void setUp() {
        service = new TrueLayerConsentService(
                "test-client-id", "test-client-secret",
                "https://auth.truelayer-sandbox.com", null);
    }

    @Test
    void generateAuthLink_returnsUrlWithState() {
        AuthLink link = service.generateAuthLink("user-1",
                List.of(ConsentScope.ACCOUNTS, ConsentScope.BALANCE),
                "https://app.example.com/callback");
        assertThat(link.authorizationUrl()).contains("state=");
        assertThat(link.authorizationUrl()).contains("client_id=test-client-id");
    }

    @Test
    void exchangeCode_unknownState_throws() {
        assertThatThrownBy(() -> service.exchangeCode("auth-code", "bogus-state"))
                .isInstanceOf(InvalidStateException.class);
    }

    @Test
    void getUserToken_noConsent_returnsNull() {
        assertThat(service.getUserToken("nonexistent-user")).isNull();
    }
}
```

- [ ] **Step 3: Run test — verify it fails**

Expected: FAIL (TrueLayerConsentService doesn't exist)

- [ ] **Step 4: Implement TrueLayerConsentService**

Create `TrueLayerConsentService.java` implementing:
- State generation with `SecureRandom` (128-bit, hex-encoded)
- State storage: `ConcurrentHashMap<String, PendingConsent>` (`state → {userId, scopes, expiry}`)
- Consent storage: `ConcurrentHashMap<String, StoredConsent>` (`userId → consent`)
- Three-step `getUserToken()`: valid token → refresh → null
- Per-userId synchronized refresh with double-check pattern
- 10-minute TTL on pending states, single-use deletion

- [ ] **Step 5: Run tests — verify they pass**

- [ ] **Step 6: Add comprehensive consent tests**

Add tests for: state expiry, state single-use, token refresh lifecycle, concurrent refresh, consent revocation, scope tracking.

- [ ] **Step 7: Commit**

```bash
git add bank-truelayer/
git commit -m "feat(#106): add TrueLayerConsentService with state-based CSRF and token refresh

Refs #106"
```

## Batch 3: Platform Integration and Wiring

### Task 5: TrueLayerBankPlatform, callback endpoint, and CDI wiring

**Files:**
- Create: `bank-truelayer/src/main/java/io/casehub/connectors/bank/truelayer/TrueLayerBankPlatform.java`
- Create: `bank-truelayer/src/main/java/io/casehub/connectors/bank/truelayer/TrueLayerAccountInformation.java`
- Create: `bank-truelayer/src/main/java/io/casehub/connectors/bank/truelayer/TrueLayerPaymentInitiation.java`
- Create: `bank-truelayer/src/main/java/io/casehub/connectors/bank/truelayer/TrueLayerAuthCallback.java`
- Create: `bank-truelayer/src/main/java/io/casehub/connectors/bank/truelayer/TrueLayerBeans.java`
- Create: `bank-truelayer/src/main/resources/application.properties`
- Test: `bank-truelayer/src/test/java/io/casehub/connectors/bank/truelayer/TrueLayerBankPlatformTest.java`

**Interfaces:**
- Consumes: `TrueLayerClient` (HTTP operations)
- Consumes: `TrueLayerConsentService` (user token resolution)
- Consumes: `BankPlatform`, `AccountInformation`, `PaymentInitiation` (SPI interfaces from bank-spi)
- Produces: `TrueLayerBankPlatform` implementing `BankPlatform`

- [ ] **Step 1: Write failing test for platform capability accessors**

```java
package io.casehub.connectors.bank.truelayer;

import io.casehub.connectors.bank.spi.AccountInformation;
import io.casehub.connectors.bank.spi.PaymentInitiation;
import org.junit.jupiter.api.Test;

import static org.assertj.core.api.Assertions.assertThat;

class TrueLayerBankPlatformTest {

    @Test
    void id_returnsTruelayer() {
        var platform = new TrueLayerBankPlatform(null, null);
        assertThat(platform.id()).isEqualTo("truelayer");
    }

    @Test
    void supports_accountInformation_returnsTrue() {
        var platform = new TrueLayerBankPlatform(null, null);
        assertThat(platform.supports(AccountInformation.class)).isTrue();
    }

    @Test
    void supports_paymentInitiation_returnsTrue() {
        var platform = new TrueLayerBankPlatform(null, null);
        assertThat(platform.supports(PaymentInitiation.class)).isTrue();
    }
}
```

- [ ] **Step 2: Run test — verify it fails**

- [ ] **Step 3: Implement TrueLayerBankPlatform**

```java
package io.casehub.connectors.bank.truelayer;

import io.casehub.connectors.bank.spi.AccountInformation;
import io.casehub.connectors.bank.spi.BankPlatform;
import io.casehub.connectors.bank.spi.PaymentInitiation;
import jakarta.enterprise.context.ApplicationScoped;

@ApplicationScoped
public class TrueLayerBankPlatform implements BankPlatform {

    private final TrueLayerClient client;
    private final TrueLayerConsentService consentService;

    public TrueLayerBankPlatform(TrueLayerClient client,
                                  TrueLayerConsentService consentService) {
        this.client = client;
        this.consentService = consentService;
    }

    @Override public String id() { return "truelayer"; }

    @Override
    public AccountInformation accountInformation(String userId) {
        return new TrueLayerAccountInformation(client, consentService, userId);
    }

    @Override
    public PaymentInitiation paymentInitiation(String userId) {
        return new TrueLayerPaymentInitiation(client, consentService, userId);
    }

    @Override
    public boolean supports(Class<?> capability) {
        return capability == AccountInformation.class
            || capability == PaymentInitiation.class;
    }
}
```

- [ ] **Step 4: Implement TrueLayerAccountInformation**

Data mapping from TrueLayer DTOs → SPI model records. Consent token resolution via `consentService.getUserToken(userId)`. Throws `ConsentExpiredException` if token is null.

- [ ] **Step 5: Implement TrueLayerPaymentInitiation**

Payment request mapping (`PaymentDestination` sealed type switch), idempotency key → header, status mapping.

- [ ] **Step 6: Create TrueLayerAuthCallback**

```java
package io.casehub.connectors.bank.truelayer;

import jakarta.inject.Inject;
import jakarta.ws.rs.GET;
import jakarta.ws.rs.Path;
import jakarta.ws.rs.QueryParam;
import jakarta.ws.rs.core.Response;

import java.net.URI;

@Path("/auth/truelayer")
public class TrueLayerAuthCallback {

    @Inject TrueLayerConsentService consentService;

    @GET
    @Path("/callback")
    public Response callback(@QueryParam("code") String code,
                             @QueryParam("state") String state) {
        ConsentInfo consent = consentService.exchangeCode(code, state);
        return Response.seeOther(URI.create("/consent/success?userId=" + consent.userId()))
                       .build();
    }
}
```

- [ ] **Step 7: Create TrueLayerBeans**

```java
package io.casehub.connectors.bank.truelayer;

import io.quarkus.oidc.client.OidcClient;
import io.quarkus.oidc.client.NamedOidcClient;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.enterprise.inject.Produces;
import org.eclipse.microprofile.config.inject.ConfigProperty;

@ApplicationScoped
public class TrueLayerBeans {

    @Produces @ApplicationScoped
    public TrueLayerClient trueLayerClient(
            @NamedOidcClient("truelayer") OidcClient oidcClient,
            @ConfigProperty(name = "casehub.connectors.bank.truelayer.base-url",
                            defaultValue = "https://api.truelayer.com") String baseUrl) {
        return new TrueLayerClient(oidcClient, baseUrl);
    }
}
```

- [ ] **Step 8: Create application.properties**

```properties
# TrueLayer OIDC client — disabled by default (enabled when credentials are configured)
quarkus.oidc-client.truelayer.auth-server-url=https://auth.truelayer.com
quarkus.oidc-client.truelayer.client-id=${TRUELAYER_CLIENT_ID:}
quarkus.oidc-client.truelayer.credentials.secret=${TRUELAYER_CLIENT_SECRET:}
quarkus.oidc-client.truelayer.grant.type=client_credentials
```

- [ ] **Step 9: Run tests — verify they pass**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn -pl bank-truelayer -am clean install`
Expected: BUILD SUCCESS

- [ ] **Step 10: Commit**

```bash
git add bank-truelayer/
git commit -m "feat(#106): add TrueLayerBankPlatform, auth callback, and CDI wiring

Refs #106"
```

### Task 6: Full build verification and documentation

**Files:**
- Modify: `CLAUDE.md` (add bank-truelayer to module table)
- Modify: `docs/guides/consumer-guide.md` (BankPlatform capability usage)
- Modify: `docs/guides/contributor-guide.md` (TrueLayer provider architecture)

**Interfaces:**
- Consumes: everything from Tasks 1-5

- [ ] **Step 1: Full build**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn clean install`
Expected: BUILD SUCCESS (all modules)

- [ ] **Step 2: Update ARC42STORIES.MD**

Update L12 (Bank SPI layer): rename from "Bank Feed Platform SPI" to "Bank Platform SPI", add capability architecture description, update `@SimulationEligible(name)` from `bank-feed-platform` to `bank-platform`, update key files list.

Add L14 (new): "TrueLayer Bank Provider" with `TrueLayerBankPlatform`, `TrueLayerClient`, `TrueLayerConsentService`, `TrueLayerAuthCallback`, `TrueLayerBeans` as key files.

Add `bank-truelayer` row to §5 module structure table.

- [ ] **Step 3: Update CLAUDE.md module table**

Add `bank-truelayer` row to the module table:
```
| `bank-truelayer` | TrueLayer BankPlatform provider (AISP + PISP) |
```

Update `bank-spi` description to reflect capability architecture.

- [ ] **Step 4: Update consumer guide**

Add BankPlatform capability usage section — `accountInformation(userId)`, `paymentInitiation(userId)`, consent flow pattern.

- [ ] **Step 5: Update contributor guide**

Add TrueLayer provider architecture section — `TrueLayerClient`, `TrueLayerConsentService`, adding new bank providers.

- [ ] **Step 6: Commit**

```bash
git add CLAUDE.md docs/ ARC42STORIES.MD
git commit -m "docs(#106): update ARC42STORIES, guides, and CLAUDE.md for BankPlatform and bank-truelayer

Refs #106"
```

## References

- [2026-09-22-truelayer-bankplatform-design.md] — design spec this plan implements
- [bank-spi/BankFeedPlatform.java] — current SPI (renamed in Task 1)
- [bank-spi/BankFeedPlatformService.java] — current service (renamed in Task 1)
- [bank-spi/NoOpBankFeedPlatform.java] — current NoOp (renamed in Task 1)
- [graphql/ConnectorBankFeedApi.java] — current GraphQL API (migrated in Task 2)
- [chat-spi/ChatPlatform.java] — capability sub-interface pattern reference
- [calendar-google/GoogleCalendarPlatform.java] — provider implementation reference
- [connectors-api/HttpHelper.java] — shared HTTP client singleton
- [PP-20260607-9794cb] — shared-http-client protocol
- [PP-20260609-0c3e24] — credential-config-ownership protocol
- [PP-20260610-83747b] — paginating-client-fail-soft protocol
- [PP-20260609-e3a2bd] — SPI id() naming convention
- [GitHub #106] — focal issue
- [GitHub #107] — deferred: consent token persistence
- [GitHub #108] — deferred: payment webhooks
