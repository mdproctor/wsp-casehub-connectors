---
layout: post
title: "bank-ref: When the Pattern Absorbs the Complexity"
date: 2026-09-25
entry_type: note
subtype: diary
projects: [casehubio/connectors]
tags: [bank-platform, ref-implementation, spi, design-patterns]
---

# bank-ref: When the Pattern Absorbs the Complexity

The connectors repo has a ref implementation pattern — `chat-ref` and `calendar-ref` both follow it: a `Backend` interface for storage, an `InMemoryBackend` with `@DefaultBean` for the CDI wiring, a `RefPlatform` that delegates, and a `Beans` producer class. Add the module, pre-load some data, write tests. Straightforward.

`bank-ref` was the test of whether that pattern holds when the SPI gets structurally more complex. BankPlatform has capability sub-interfaces (`AccountInformation`, `PaymentInitiation`) with user-scoped accessors — `accountInformation(userId)` returns a capability instance, not a direct result. CalendarPlatform is flat: `listCalendars()` lives directly on the interface. ChatPlatform has capabilities too, but nine of them with a Builder pattern managing the wiring. BankPlatform sits in between — two capabilities, no Builder needed, but the accessor pattern still changes the delegation shape.

The interesting finding: it didn't change the delegation shape at all. The `BankBackend` interface stays flat — `listAccounts()`, `balance()`, `initiatePayment()` — because the userId parameter is a domain concern of real providers (consent tokens, per-user scoping under PSD2), not of the storage abstraction. The ref implementation accepts `userId` on the capability accessors and ignores it. `RefAccountInformation` and `RefPaymentInitiation` are thin wrappers that forward to the backend. They exist to satisfy the type system, not to add behaviour.

The payment lifecycle was the one design question with real options. Real providers use webhook-driven state changes or time-based progression. We went with auto-advance on poll: `initiatePayment()` returns `AUTHORIZATION_REQUIRED`, each `paymentStatus()` call advances to the next state (→ `EXECUTED` → `SETTLED`), then stays terminal. Deterministic, trivially testable, no timing sensitivity. A `PaymentState` inner class with a three-element progression array is the entire state machine.

The pre-loaded data is deliberately mundane — three UK bank accounts (current, savings, credit card) and twelve transactions across a month of consumer spending. Tesco Express, Shell Garage, Netflix, British Gas. The point isn't interesting data; it's data that looks real enough that a developer testing their code against the SPI gets a feel for what production data shapes look like. Amounts are `BigDecimal` throughout, directions are `DEBIT`/`CREDIT`, one transaction is `PENDING` to cover both status values.

The pattern held. Backend/InMemory/Ref/Beans, same shape as calendar-ref, same CDI tier structure, same test style. The capability sub-interfaces added two package-private wrapper classes and zero architectural decisions. That's the mark of a good pattern — it absorbs structural variation without changing its own shape.
