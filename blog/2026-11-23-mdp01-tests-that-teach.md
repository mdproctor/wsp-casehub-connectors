---
title: Tests That Teach
date: 2026-11-23
author: Mark Proctor
tags: [playbooks, tutorials, testing, connectors]
---

A test that asserts `succeeded: true` tells you one thing: it didn't crash. A test that asserts `balance.amount: 3245.67` tells you what the connector returns. The first is a gate. The second is documentation.

We've been writing playbook tutorials for every connector SPI — one per domain, twelve in total, each meant to teach a reader the integration pattern by reading the test file alone. The first pass had assertions like `not-null: true` on every field. Technically correct, completely useless as teaching material. A reader learns nothing from "the name exists."

The fix was structural matching. `matches:` in `assertThat` checks the subject's fields when the expected value is a map:

```yaml
- assertThat: run.captures.balance
  matches:
    accountId: acc-100
    currency: GBP
    amount: 3245.67
    current: 3245.67
```

One assertion, the whole response shape. A reader sees what `bank-balance` returns — account ID, currency, two amount fields — without reading the SPI Javadoc. The test *is* the documentation.

The location tutorial is the most interesting. It chains three steps: get my device position, search for restaurants nearby, get details of the nearest one. The test reads like a story:

```yaml
name: device position near Balham, two restaurants within 2km
---
- run-playbook: 03-find-nearby-places
  capture: run
- assertThat: run.captures.myLocation.location
  matches:
    lat: 51.443
    lng: -0.1525
- assertThat: run.captures.nearby.items
  hasSize: 2
- assertThat: run.captures.place
  matches:
    name: The Crown Balham
    formattedAddress: "102 Balham High Road, London SW12 9AA"
    rating: 4.1
```

Device says I'm near Balham. Two restaurants within 2km. The Crown is closest — 4.1 stars, 102 Balham High Road. All from ref providers, no credentials needed. `mvn test` runs it.

The DeviceLocation capability made this possible. Without it, the tutorial hardcoded coordinates — `lat: 51.4430, lng: -0.1525` — and the reader had to guess what those numbers meant. With it, the playbook says "where am I?" and the reader follows the chain: position → search → details.

Scaling from three tutorials to twelve needed infrastructure work. The graphql module generates step-type plugins for every domain, but each plugin needs its backing API service wired for tests. `PlaybookApiServices` now has an inner class per domain — Calendar, Commerce, Contacts, Document, Health, Travel — each constructing a ref backend and setting the right identity. Two patterns here: some APIs inject `SecurityIdentity` (Commerce, Contacts, Health, Travel), others inject `CurrentPrincipal` (Calendar, Document, Email). Same user, different interface. A shared `TEST_PRINCIPAL` handles the second; a shared `QuarkusSecurityIdentity` builder handles the first.

One gap surfaced: the YAML step engine can't coerce a string into a `LocalDate`. The travel search step takes `@Required LocalDate date`, and there's no mechanism to convert `date: 2026-12-20` from YAML into the Java type. The travel tutorial works around it by looking up an accommodation by ID instead of searching by date. The search would be a better tutorial, but it needs a type coercion extension in the step engine first.

The existing tests — voice echo, banking assistant, joint account verification, dinner booking — were still using relative paths and weak assertions. We migrated them to match: stem references instead of `../../../main/playbooks/...` traversals, structural assertions instead of `not-null: true`. The pattern is now universal. Every playbook test in the module follows the same conventions.

Four conventions keep the tutorials clean:

**Bare-list format.** Playbooks use `---` instead of `steps:`. The step list reads like instructions, not a data structure.

**Stem resolution.** Tests reference playbooks by stem — `run-playbook: 02-check-bank-balance` — not by relative path. The test runner derives the package from the test file's location and finds the playbook on the classpath.

**Structural matching.** `matches:` with a map asserts the shape in one block. Literal values exact-match; map values apply matchers (`{ greaterThan: 3000 }`). No more four-line assertion blocks for a single balance check.

**Error guards.** Every test checks `run.error equals: ""` before asserting on captures. When a step fails, the test reports the actual error message — not a confusing null on some downstream field.

The tutorials are `tutorials/` in the playbooks module. The composed scenarios (dinner booking, banking assistant, joint account verification) stay alongside in their own directories. Tutorials are recipes — one domain, one pattern. Scenarios show composition.
