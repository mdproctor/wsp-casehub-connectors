---
layout: post
title: "Search Slots Into Place"
date: 2026-10-06
entry_type: note
subtype: diary
projects: [casehubio/connectors]
tags: [email, spi, search, simulation]
---

# Search Slots Into Place

The EmailPlatform SPI had no search method. Every other platform that
needed search — DocumentPlatform, ContactsPlatform, ProjectPlatform —
already had one. Email was the holdout, identified during the simulation
integration audit back in #139.

What made this interesting wasn't the implementation. It was how little
design work remained by the time we got here. The #138 design spec had
already classified email search as "interpretive" and a `search-corpus.yaml`
had shipped with #143's seed data work — sitting in `email-spi/src/main/resources/simulation/email/`,
wired to nothing. The interface gap was the only thing missing.

The method signature follows the cross-SPI pattern: `Page<EmailSummary> search(String query, PageRequest pagination)`.
No `userId` parameter — EmailPlatform is flat, unlike the capability-sub-interface
SPIs that scope operations per user. The ref implementation does naive
case-insensitive substring matching across subject, from, and body text.
Nothing clever, but consistent with how every other ref does search.

The Gmail implementation was the satisfying part. Gmail's `messages.list`
accepts a `q` parameter that supports the full Gmail search syntax natively —
`from:alice`, `has:attachment`, `after:2026/01/01`. We pass the query straight
through. The only wrinkle: search results span all mailboxes, so the
`mailboxId` on each result needs deriving from the message's labels rather than
being passed in as a parameter. A `primaryLabel()` helper on `GmailMessageMapper`
handles that — filtering out metadata labels (UNREAD, STARRED, IMPORTANT,
category labels) and taking the first remaining label, defaulting to INBOX.

Wiring the corpus into `simulation.yaml` was a single YAML block — `seq` strategy
with `WRAP` exhaustion, matching the document-spi pattern exactly.

The session that planted the corpus file and classified the search method
did the harder work. This session just filled the slot.
