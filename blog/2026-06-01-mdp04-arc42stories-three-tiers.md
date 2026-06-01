---
layout: post
title: "Arc42Stories Needs Three Tiers"
date: 2026-06-01
type: phase-update
entry_type: note
subtype: diary
projects: [casehub-connectors]
tags: [arc42stories, architecture, planning]
---

The build came up clean first pass — all 17 email-inbound tests through, no IDLE
timing failures. After two sessions of coaxing GreenMail into behaving, that counts
as a result.

## The Profile Gap

The plan was to file the arc42stories migration issue for connectors — look at what
devtown did, figure out the equivalent for a library, write the issue. Straightforward.

Then I actually read arc42stories-casehub-profile.md. I'd assumed from context that
foundation modules just use the base spec directly, no profile. The profile says it
applies to casehub-aml, casehub-clinical, casehub-devtown, casehub-life,
casehub-drafthouse, QuarkMind — confirming the assumption. But reading it carefully,
the profile carries two distinct things. The Foundation Layer Taxonomy (Domain baseline
→ work → qhorus → ledger → engine → trust routing) is genuinely harness-app-specific —
it describes how apps integrate the foundation, not how the foundation documents itself.
Everything else — the artifact schema, §8 crosscutting convention references, the
anti-pattern inline requirement, the platform references for §3 — applies equally to a
foundation module writing its own ARC42STORIES.MD.

That gap turned into parent#132: extend the spec with a three-tier taxonomy.

**Application** — systems integrating foundation modules progressively. Harness apps.
Many layers, Chapters = foundation module integrations.

**Foundation** — core infrastructure. Fewer layers. Chapters = capability increments,
not foundation-module integrations. connectors, work, qhorus, ledger, engine all sit
here.

**Extension** — optional building blocks that orbit foundation or application. Pluggable,
less central.

"Ecosystem" was the first word I reached for and it's wrong — ecosystem means the whole
surrounding community, not a level within it. "Core" doesn't work as a tier name either;
it already does too many jobs (submodule naming convention, informal shorthand for the
most critical modules). "Extension" is precise.

connectors#13 (migrate DESIGN.md → ARC42STORIES.MD) can proceed on foundation-tier
assumptions now. The spec update formalises it later.

## Five Branches and a Zombie

Branch hygiene: five branches stamped closed, main pushed to origin (two commits had
been sitting unshipped since the IMAP work), one worktree cleaned up. The worktree's
lock turned out to be held by a Claude process that had been running for 97 hours —
killed it, removed the lock, done.

The five closed branches were all squash-merged work with no unique content: issues 4,
6, 7, 9, and 11. Each branch's code was on main; the branches just hadn't been stamped.
