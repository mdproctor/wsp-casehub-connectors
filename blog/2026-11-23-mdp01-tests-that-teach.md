---
title: Tests That Teach
date: 2026-11-23
author: Mark Proctor
tags: [playbooks, tutorials, testing, connectors]
---

A test that asserts `succeeded: true` tells you one thing: it didn't crash. A test that asserts `balance.amount: 3245.67` tells you what the connector returns. The first is a gate. The second is documentation.

We've been writing playbook tutorials for the connector SPIs — small how-to examples, one per domain, each meant to teach a reader the integration pattern by reading the test file alone. The first pass had assertions like `not-null: true` on every field. Technically correct, completely useless as teaching material. A reader learns nothing from "the name exists."

The fix was structural matching. Platform #904 added `matches:` to `assertThat` — when the expected value is a map, it checks the subject's fields:

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

The DeviceLocation capability (#334) made this possible. Without it, the tutorial hardcoded coordinates — `lat: 51.4430, lng: -0.1525` — and the reader had to guess what those numbers meant. With it, the playbook says "where am I?" and the reader follows the chain: position → search → details.

Three conventions made the tutorials clean:

**Bare-list format.** Playbooks use `---` instead of `steps:`. Four fewer characters, one less indentation level, but more importantly: the step list reads like instructions, not a data structure.

**Stem resolution.** Tests reference playbooks by stem — `run-playbook: 02-check-bank-balance` — not by relative path. The test runner derives the package from the test file's location and finds the playbook on the classpath. The reference reads like a name, not a filesystem traversal.

**Structural matching.** `matches:` with a map asserts the shape in one block. Literal values exact-match; map values apply matchers (`{ greaterThan: 3000 }`). No more four-line assertion blocks for a single balance check.

The tutorials are `tutorials/` in the playbooks module. The composed scenarios (dinner booking, banking assistant, joint account verification) stay alongside in their own directories. Tutorials are recipes — one domain, one pattern. Scenarios show composition.
