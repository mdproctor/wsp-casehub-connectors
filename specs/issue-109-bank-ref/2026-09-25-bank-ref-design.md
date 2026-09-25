# bank-ref: In-Memory Reference BankPlatform — Design Spec

> **Issue:** casehubio/connectors#109
> **Date:** 2026-09-25
> **Status:** Draft
> **Decisions:** [decisions.md](decisions.md)

## Overview

New `bank-ref` module providing `RefBankPlatform` — an in-memory
`BankPlatform` implementation with pre-loaded realistic test data.
Follows the established pattern from `chat-ref` and `calendar-ref`:
Backend interface → InMemoryBackend → RefPlatform → Beans producer.

Supports both `AccountInformation` and `PaymentInitiation` capabilities.
Works out of the box with zero configuration.

## Architecture

### Module structure

```
bank-ref/
  src/main/java/io/casehub/connectors/bank/ref/
    BankBackend.java              — storage operations interface
    InMemoryBankBackend.java      — @DefaultBean ConcurrentHashMap impl + pre-loaded data
    RefBankPlatform.java          — BankPlatform impl, delegates to backend
    RefAccountInformation.java    — AccountInformation impl
    RefPaymentInitiation.java     — PaymentInitiation impl
    BankRefBeans.java             — CDI producer
  src/test/java/io/casehub/connectors/bank/ref/
    RefBankPlatformTest.java      — unit tests
```

### Component diagram

```
┌─────────────────────────────────────────────────┐
│  bank-ref module                                │
│                                                 │
│  BankRefBeans (@Produces)                       │
│    └─→ RefBankPlatform ──→ BankBackend          │
│           ├─ RefAccountInformation              │
│           └─ RefPaymentInitiation               │
│                                                 │
│  InMemoryBankBackend (@DefaultBean)             │
│    └─ implements BankBackend                    │
│    └─ pre-loaded accounts, txns, balances       │
│    └─ payment state machine                     │
└─────────────────────────────────────────────────┘
         │
         ▼ implements
┌─────────────────────┐
│  bank-spi           │
│  BankPlatform       │
│  AccountInformation │
│  PaymentInitiation  │
└─────────────────────┘
```

## BankBackend interface

Defines the storage operations that `RefBankPlatform` delegates to.
Follows the `ChatBackend` / `CalendarBackend` pattern.

```java
public interface BankBackend {

    List<AccountInfo> listAccounts();

    AccountBalance balance(String accountId);

    Page<Transaction> listTransactions(String accountId,
                                       Instant from, Instant to,
                                       PageRequest pagination);

    Transaction getTransaction(String accountId, String transactionId);

    InitiatedPayment initiatePayment(PaymentRequest payment);

    PaymentStatus paymentStatus(String paymentId);
}
```

No `userId` parameter on any method — the backend holds a single shared
dataset (D1). The `RefBankPlatform` accepts `userId` on the capability
accessors (required by the SPI) but does not pass it through.

## InMemoryBankBackend

`@DefaultBean @ApplicationScoped`. ConcurrentHashMap-backed. Pre-loaded
with realistic UK test data on construction.

### Pre-loaded accounts

| ID | Name | Type | Currency |
|----|------|------|----------|
| `acc-100` | Current Account | CURRENT | GBP |
| `acc-200` | Savings Account | SAVINGS | GBP |
| `acc-300` | Credit Card | CREDIT_CARD | GBP |

### Pre-loaded balances

| Account | Available | Current | Currency |
|---------|-----------|---------|----------|
| `acc-100` | 2,450.00 | 2,450.00 | GBP |
| `acc-200` | 15,000.00 | 15,000.00 | GBP |
| `acc-300` | 4,750.00 | 5,000.00 | GBP |

### Pre-loaded transactions

12 transactions across `acc-100` (current account), spanning a realistic
month of UK consumer spending. All amounts positive per SPI convention,
direction indicates debit/credit:

| ID | Account | Amount | Dir | Merchant | Category | Date | Status |
|----|---------|--------|-----|----------|----------|------|--------|
| `txn-001` | acc-100 | 3.50 | DEBIT | Tesco Express | Groceries | 2026-09-01 | BOOKED |
| `txn-002` | acc-100 | 45.00 | DEBIT | Shell Garage | Transport | 2026-09-02 | BOOKED |
| `txn-003` | acc-100 | 2,500.00 | CREDIT | ACME Corp | Salary | 2026-09-03 | BOOKED |
| `txn-004` | acc-100 | 12.99 | DEBIT | Netflix | Entertainment | 2026-09-05 | BOOKED |
| `txn-005` | acc-100 | 67.50 | DEBIT | Sainsburys | Groceries | 2026-09-07 | BOOKED |
| `txn-006` | acc-100 | 150.00 | DEBIT | British Gas | Utilities | 2026-09-10 | BOOKED |
| `txn-007` | acc-100 | 8.90 | DEBIT | Costa Coffee | Dining | 2026-09-12 | BOOKED |
| `txn-008` | acc-100 | 35.00 | DEBIT | Amazon UK | Shopping | 2026-09-14 | BOOKED |
| `txn-009` | acc-100 | 500.00 | DEBIT | Nationwide BS | Transfers | 2026-09-15 | BOOKED |
| `txn-010` | acc-100 | 22.50 | DEBIT | Deliveroo | Dining | 2026-09-18 | BOOKED |
| `txn-011` | acc-100 | 9.99 | DEBIT | Spotify | Entertainment | 2026-09-20 | BOOKED |
| `txn-012` | acc-100 | 75.00 | DEBIT | TfL | Transport | 2026-09-22 | PENDING |

### Pagination

`listTransactions()` applies offset-based pagination internally (no
cursor complexity for an in-memory dataset). Filters by date range
first, then paginates the result. Returns `Page.of(items)` when all
items fit in one page. For multi-page results, uses a numeric cursor
(string representation of the offset).

### Payment state machine (D2)

`initiatePayment()` stores the payment and returns it with status
`AUTHORIZATION_REQUIRED`. Each `paymentStatus()` call advances the
stored state:

```
AUTHORIZATION_REQUIRED → EXECUTED → SETTLED (terminal)
```

Once SETTLED, subsequent calls return SETTLED. The `hostedPaymentPageLink`
in `InitiatedPayment` is a placeholder URL
(`https://ref.bank.example/pay/{paymentId}`).

Idempotency: if `initiatePayment()` is called with a previously-seen
`idempotencyKey`, returns the existing payment (same as real providers).

### Error handling

- `balance(unknownId)` → `NoSuchElementException`
- `getTransaction(accountId, unknownTxnId)` → `NoSuchElementException`
- `paymentStatus(unknownId)` → `NoSuchElementException`

Consistent with the SPI error contract.

## RefBankPlatform

Plain class (no CDI annotations). Implements `BankPlatform`. Delegates
to `BankBackend`.

```java
public class RefBankPlatform implements BankPlatform {

    private final BankBackend backend;

    public RefBankPlatform(BankBackend backend) {
        this.backend = backend;
    }

    @Override public String id() { return "ref"; }

    @Override
    public AccountInformation accountInformation(String userId) {
        return new RefAccountInformation(backend);
    }

    @Override
    public PaymentInitiation paymentInitiation(String userId) {
        return new RefPaymentInitiation(backend);
    }

    @Override
    public boolean supports(Class<?> capability) {
        return capability == AccountInformation.class
            || capability == PaymentInitiation.class;
    }
}
```

`userId` is accepted but not used (D1). Both capabilities are always
supported.

## RefAccountInformation / RefPaymentInitiation

Thin wrappers that delegate to `BankBackend`:

```java
class RefAccountInformation implements AccountInformation {
    private final BankBackend backend;
    // delegates listAccounts, balance, listTransactions, getTransaction
}

class RefPaymentInitiation implements PaymentInitiation {
    private final BankBackend backend;
    // delegates initiatePayment, paymentStatus
}
```

Package-private — not part of the public API.

## CDI wiring

```java
@ApplicationScoped
public class BankRefBeans {

    @Produces
    @ApplicationScoped
    public RefBankPlatform refBankPlatform(BankBackend backend) {
        return new RefBankPlatform(backend);
    }
}
```

Follows `ChatRefBeans` / `CalendarRefBeans` pattern. The produced
`RefBankPlatform` is a normal (non-default) bean. When bank-ref is on
the classpath:

- `NoOpBankPlatform` (`@DefaultBean`) is suppressed
- `RefBankPlatform` registers as `BankPlatform` with id `"ref"`
- If `bank-truelayer` is also present, both coexist —
  `BankPlatformService` routes by id

## Maven module

```xml
<artifactId>casehub-connectors-bank-ref</artifactId>
```

Dependencies: `casehub-connectors-bank-spi`, `quarkus-arc`.
Test dependencies: `quarkus-junit`, `assertj-core`.
Plugins: `jandex-maven-plugin`, `maven-jar-plugin` (test-jar).

Added to parent pom `<modules>` after `bank-spi`, before
`bank-truelayer`.

## Testing

`RefBankPlatformTest` — direct unit tests (no Quarkus test profile):

### AccountInformation tests
- `id()` returns `"ref"`
- `listAccounts()` returns 3 accounts with correct types and currencies
- `balance(existingId)` returns non-null balance with correct currency
- `balance(unknownId)` throws `NoSuchElementException`
- `listTransactions()` with full date range returns all 12 transactions
- `listTransactions()` with narrow date range filters correctly
- `listTransactions()` with pagination returns correct page size and cursor
- `getTransaction(existingId)` returns correct transaction
- `getTransaction(unknownId)` throws `NoSuchElementException`
- `supports(AccountInformation.class)` returns true
- `supports(PaymentInitiation.class)` returns true

### PaymentInitiation tests
- `initiatePayment()` returns AUTHORIZATION_REQUIRED with payment ID and hosted link
- `paymentStatus()` first call returns EXECUTED
- `paymentStatus()` second call returns SETTLED
- `paymentStatus()` third+ call stays at SETTLED
- `paymentStatus(unknownId)` throws `NoSuchElementException`
- Idempotent payment — same idempotencyKey returns same payment

### userId handling (D1)
- `accountInformation("user-a")` and `accountInformation("user-b")` return same data

## Deliverables beyond code

### CLAUDE.md update
- Add `bank-ref` to module table: "In-memory reference BankPlatform"
- Update project description to include bank-ref

### ARC42STORIES.MD update
- §5 Module structure table: add `bank-ref` row
- §9 Update L12 (Bank Platform SPI) to reference bank-ref as available implementation

### Consumer guide update
- `docs/guides/consumer-guide.md`: add bank-ref usage section with
  dependency snippet, zero-config explanation, and test example

## References

- `chat-ref/RefChatPlatform.java` — ref implementation pattern
- `chat-ref/ChatBackend.java` — backend interface pattern
- `chat-ref/InMemoryChatBackend.java` — in-memory storage pattern, `@DefaultBean` wiring
- `chat-ref/ChatRefBeans.java` — CDI producer pattern
- `calendar-ref/RefCalendarPlatform.java` — simpler ref implementation reference
- `calendar-ref/CalendarRefBeans.java` — CDI producer pattern
- `calendar-ref/pom.xml` — Maven module structure
- `bank-spi/BankPlatform.java` — SPI interface being implemented
- `bank-spi/AccountInformation.java` — AISP capability interface
- `bank-spi/PaymentInitiation.java` — PISP capability interface
- `bank-spi/NoOpBankPlatform.java` — `@DefaultBean` fallback being replaced
- `bank-spi/BankPlatformService.java` — routing by id (coexistence model)
- `connectors-api/Page.java`, `connectors-api/PageRequest.java` — pagination types
- Issue #106 design spec — BankPlatform SPI definition
- D1 (decisions.md) — userId handling: ignore, shared data
- D2 (decisions.md) — payment lifecycle: auto-advance on poll
