---
layout: post
title: "The Library Gets Its Memory"
date: 2026-06-17
type: phase-update
entry_type: note
subtype: diary
projects: [casehub-connectors]
tags: [arc42stories, documentation, architecture]
series: issue-21-arc42stories
---

*Part of a series on [#21 — docs: create ARC42STORIES.MD](https://github.com/casehubio/connectors/issues/21).*

casehub-connectors had a DESIGN.md. It described the SPIs, the module structure,
the data model, the configuration. It was accurate. It was also flat — it told
you what the library is, not what it took to build it, why the decisions were
made, or how to replicate it in a different context.

That gap has a name now: ARC42STORIES.MD.

## The source material problem

Writing an architecture document is usually one of two things: you write it before
the code exists, which means you're speculating, or you write it after, which means
you're summarising from memory and probably losing the interesting parts.

We had a third option. Eleven diary entries, eight design specs, three ADRs, and a
DESIGN.md that was current as of last week. The raw material was richer than anything
I'd usually start from. The challenge wasn't discovery — it was synthesis.

The five chapters mapped directly to the delivery sequence: outbound SPI first,
then inbound, then IMAP specifically, then the MCP surface, then the Slack bot.
Each chapter introduced at least one new layer. That structure wasn't designed —
it emerged from reading the commit history in sequence and asking what each phase
had actually delivered to the caller.

The five layers followed from the same reading. L1 through L5 aren't a theoretical
taxonomy; they're a record of what had to exist before the next thing could be built.

## What the format demands

The part that took the most thought was the "Pattern to replicate" sections.

ARC42STORIES asks you to write numbered steps that an LLM — or a developer in a
completely different domain — could follow to implement the same layer from scratch.
That's a harder question than "what does this layer do." It forces you to articulate
the decisions that aren't visible in the code: why the SPI uses a string id rather
than a type token, why `ConnectorMeshBridge` has `@Unremovable`, why `parsePage()`
returns a record rather than throwing.

Writing those sections from diary entries was natural. mdp07 — the one I titled
"what DESIGN.md doesn't know" — was practically a first draft of the L1 pattern.
The diary captures intent at the time of implementation. The ARC42STORIES distils
it into something replicable.

## The stale reference catch

One thing I'd flagged in the issue: verify §12 issue references before publishing.
When we checked `qhorus#249`, it had closed on 2026-06-12 — while the branch was
in progress. The HANDOFF had it as open; the document would have claimed it was
still pending.

That's the kind of drift that accumulates invisibly. It's also exactly the scenario
the verification step exists to catch. We updated the §12 entry to reflect the
correct state before committing.

## What changes now

The practical effect is that `java-update-design` now routes to `ARC42STORIES.MD §10`
when it picks up architectural decisions. Previously it targeted DESIGN.md — which had
no equivalent section. Future sessions check the document for drift after SPI and
connector changes.

DESIGN.md stays for now. It's referenced as the source of the §9 build roadmap, and
retiring it cleanly is a separate step. But it's no longer the primary reference.
