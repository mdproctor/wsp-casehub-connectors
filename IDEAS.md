# Idea Log

Undecided possibilities — things worth remembering but not yet decided.
Promote to an ADR when ready to decide; discard when no longer relevant.

---

## 2026-06-07 — Quarkus demo chat service (Slack-like)

**Priority:** medium
**Status:** active

A lightweight Quarkus service for demos: WebSocket-based chat UI (served as a static resource), REST endpoint for agents to POST messages, real-time display in browser. No Docker, no external accounts — launches with `quarkus dev`.

**Context:** Running a live Slack server for demos is friction-heavy. This gives the same visual story (agents interacting in a chat channel) with zero infrastructure, and would naturally host a `DemoChatConnector` in the connectors library for demo/test use.

**Promoted to:**

---

## 2026-06-07 — CaseHub-native chat substrate

**Priority:** low
**Status:** active

A first-class, CaseHub-owned channel model with persistent rooms, participants, message history, and thread context — not just delivery to third-party systems. Would underpin Claudony's channel view and DraftHouse's direction as they converge on chat-room-like functionality tailored to CaseHub's cases and channels.

**Context:** Claudony's side channel view and DraftHouse's trajectory are pointing toward a need for something custom. The blocker is adoption cost — pulling people off Slack/Teams is high friction unless CaseHub's unique use cases clearly justify it. Park until that case becomes undeniable.

**Promoted to:**
