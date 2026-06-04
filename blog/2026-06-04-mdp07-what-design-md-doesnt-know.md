---
layout: post
title: "What DESIGN.md Doesn't Know"
date: 2026-06-04
type: phase-update
entry_type: note
subtype: diary
projects: [casehub-connectors]
tags: [arc42stories, architecture, documentation]
---

The session started without a handover file. The workspace existed — blog
entries, ADRs, design journal, specs — but nothing pointing at what came next.
I reconstructed from `git log` and the open issue list: three open issues,
nine closed, a delivery arc visible in the commit messages. Enough.

The work was the architecture document migration: DESIGN.md into ARC42STORIES.MD.
The bootstrap existed with all twelve sections present, most marked 🔲 pending.

For a foundation-tier module, the interesting question was §9 Journeys and
Chapters. The CasHub profile is written primarily for application-tier harness
apps, where Chapters map to foundation-module integration layers — you add
casehub-work, then casehub-qhorus, then casehub-ledger, each Chapter a new
foundation dependency. Connectors doesn't integrate those modules; it is one
of them. The chapter taxonomy had to be different.

I settled on capability increments: C1 for the outbound SPI and five built-in
transports, C2 for inbound transport (two SPI types, four webhook implementations,
IMAP IDLE with attachments), C3 for the MCP tool surface. Each Chapter is a
coherent deliverable someone could ship and use independently.

What made the reconstitution tractable was the blog entries. DESIGN.md has the
what — module structure, SPI contracts, data model, configuration. The blogs have
the why. Why IMAP IDLE only with no polling fallback: ADR-0002 gives the decision;
mdp03 gives the reasoning that led there. Why `ConnectorMeshBridge` lives in `core`
and not `mcp`: ADR-0003 records the choice; mdp06 explains the context that made
it obvious. The three inline anti-patterns in §8 — `@Observes` instead of
`@ObservesAsync`, blocking observer, omitting `@Unremovable` — appear somewhere
in the session writing, not in DESIGN.md. A clean architecture document needs
both layers.

The §6 runtime scenarios were the one section that required synthesis. The sequence
diagrams — outbound CDI delivery, Slack webhook receipt, MCP tool call, IMAP IDLE —
don't exist anywhere in the source material in that form. We derived them from the
DESIGN.md contracts and the blog narratives.

One find during wrap: project main was eight commits ahead of origin, all from the
previous session's MCP work, sitting unshipped. Pushed.
