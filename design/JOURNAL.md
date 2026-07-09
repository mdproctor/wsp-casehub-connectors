# Design Journal — issue-61-qhorus-chat-ui

## §9.1 — Qhorus Chat UI: Conversation Model and Component Architecture

**Date:** 2026-07-07 — 2026-07-09

### Decision: Qhorus-native data model, not generic chat

The chat UI renders qhorus messages with full structural awareness —
speech acts, correlation chains, commitment lifecycles — not just flat
text. The data model mirrors qhorus's `MessageType` (9 speech acts),
`ActorType` (HUMAN/AGENT/SYSTEM), and `CommitmentState` (7 states).
A backward-compatible adapter maps simpler backends (chat-demo WebSocket)
to the same types with sensible defaults (messageType=EVENT, actorType=HUMAN).

**Why:** The primary use case is observing agent-to-agent communication in
claudony, where the speech act structure IS the value. Flattening to
Slack-like text loses the information that makes agent conversations
comprehensible.

### Decision: Component family (primitives → composites → workbench)

Three layers, each independently usable:
- Primitives (LitElement): qhorus-message, reaction-bar, thread, message-input
- Composites (LitElement + dataset pipeline): channel-feed, channel-nav, member-panel
- Workbench: assembles with pages split/dockBar, owns data connection

**Why:** Embeddability. Claudony mounts the full workbench. A case detail
page embeds just the feed. A dashboard shows just the channel nav. The
notification-inbox pattern in blocks-ui validated this approach.

### Decision: Space → Channel → Topic → Thread hierarchy

Designed after research across 11 chat platforms and 6 agent frameworks.
Key finding: Zulip's mandatory topic model is the strongest structural
pattern for agent communication. Topics are named, persistent sub-
conversations — not anonymous reply chains. Threads (correlation chains)
nest within topics.

Qhorus model enrichments filed as qhorus#328 (6 child issues: #329–#334).
Phase 1 works without these; Phases 3–4 depend on them.

### Decision: Build alongside, swap atomically

New code in `src/qhorus/` alongside old code until ready. Single commit
swaps the entry point and deletes all old files. No feature flags, no
conditional rendering, no period of mixed old/new.
