---
title: "BankPlatform Gets Real — TrueLayer as the First Provider"
date: 2026-09-23
author: mdp
entry_type: note
subtype: diary
projects: [casehubio/connectors]
series: issue-106-truelayer-bankfeed
tags: [banking, open-banking, psd2, truelayer, spi, capability-pattern]
---

# BankPlatform Gets Real — TrueLayer as the First Provider

The BankFeedPlatform SPI shipped with issue #94 as a flat interface — `listAccounts()`, `balance()`, `listTransactions()`. Read-only, no provider implementation, pure contract. It was designed to work with the simulation framework and nothing else. Today we implemented the first real provider, and the SPI didn't survive contact with reality.

## The naming problem that surfaced the design problem

The original name — BankFeedPlatform — told a lie. "Feed" implies a push stream. This SPI is pull-based query. But the rename question opened a bigger one: if we're building a real banking provider, should it only read data? TrueLayer offers both account information (AISP) and payment initiation (PISP). An "account" platform that can't make payments is artificially constrained.

The temptation was two separate SPIs: `BankAccountPlatform` for reads, `BankPaymentPlatform` for writes. Clean separation, obvious naming. But from the domain — banking is one domain with multiple regulated capabilities, not two separate domains. PSD2 structures access that way (AISP and PISP are separately licensed), and the SPI should respect that distinction. But it should respect it through capability sub-interfaces, not through separate SPIs.

So `BankFeedPlatform` became `BankPlatform` with `AccountInformation` and `PaymentInitiation` capabilities — same pattern as ChatPlatform. The simulation framework's recursive wrapper generation (platform#375) handles the interception.

## Where the design review earned its keep

The brainstorming decisions went through a light review that caught five issues. Three of them were architectural mistakes I'd have shipped:

**Consent on the SPI.** I'd put `generateAuthLink()` and `exchangeCode()` on BankPlatform as a capability. The reviewer pointed out that consent is a precondition for banking operations, not a banking operation itself — like putting `login()` on every domain service. Consent became provider-internal infrastructure: `TrueLayerConsentService` handles the OAuth2 flow, and the SPI stays focused on accounts and payments.

**Webhook for the consent callback.** I routed the TrueLayer consent redirect through `WebhookInboundConnector`. The reviewer verified that `WebhookRouter` can only return 200 OK — it has no redirect result type. OAuth2 consent callbacks need a 302 redirect back to the application. Different pattern entirely. Became a dedicated JAX-RS endpoint.

**Implicit request-scoped context.** I proposed a `@RequestScoped` `BankContext` CDI bean to carry per-user consent tokens implicitly. The reviewer checked — ChatPlatform has no implicit context. Every method takes parameters explicitly. And `@RequestScoped` breaks outside HTTP requests (scheduled tasks, event handlers). The solution was simpler: user-scoped capability accessors. `accountInformation(userId)` is a domain concern ("whose accounts?"), and the provider resolves the consent token from its internal store.

The spec review then ran three rounds and caught the `@NamedOidcClient` qualifier (without it, CDI silently resolves to the default OIDC client), the access token refresh mechanism (without it, users re-consent every hour instead of every 90 days), and the state parameter keying inversion for CSRF protection.

## What's there now

`bank-spi` has the capability-based `BankPlatform` with `AccountInformation` (AISP) and `PaymentInitiation` (PISP). User-scoped capability accessors. `PaymentDestination` as a sealed hierarchy (UK sort code/account number now, IBAN/BIC when SEPA support arrives — the sealed type forces exhaustive switches).

`bank-truelayer` has four main classes. `TrueLayerClient` handles HTTP calls to TrueLayer's Data API and Payments API using `HttpHelper.CLIENT` with `jakarta.json` parsing. `TrueLayerConsentService` manages the PSD2 consent lifecycle — state-based CSRF, token storage, three-step token resolution (valid → refresh → re-consent), per-userId synchronized refresh. `TrueLayerBankPlatform` wires the two together through the SPI's capability interfaces. `TrueLayerAuthCallback` is the JAX-RS consent redirect endpoint.

Client credentials use `quarkus-oidc-client` with a named configuration (`@NamedOidcClient("truelayer")`) — Quarkus manages token caching, refresh, and thread safety for machine-to-machine auth. User consent tokens are managed by `TrueLayerConsentService` separately, because PSD2 consent is per-user and per-scope.

## What's deferred

Consent token persistence is in-memory (`ConcurrentHashMap`). A JVM restart loses all consent tokens, forcing every user through re-consent (bank redirect, SCA). For dev and test that's fine. For production, it needs a database — tracked as #107.

Payment status webhooks are polling-only via `paymentStatus()`. TrueLayer sends webhook notifications for payment lifecycle events, but adding a webhook receiver (signature validation, CDI event firing) is a separate concern — tracked as #108.

## The capability-pattern implication

This is the second capability-based SPI in connectors (after ChatPlatform) and the first to use user-scoped capability accessors. The `accountInformation(userId)` pattern is specific to PSD2 — each user grants their own consent, so the provider needs to know whose accounts are being queried. ChatPlatform doesn't need this because chat operations are per-platform, not per-user.

If a third regulated SPI appears (insurance, health records — anything with per-user consent), the user-scoped accessor pattern is now proven. And the simulation framework's recursive wrapper generation handles the interception transparently, so there's no decorator ceremony to repeat.
