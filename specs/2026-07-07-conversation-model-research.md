# Conversation Model Research

**Date:** 2026-07-07
**Purpose:** Background research informing the conversation model design
in [qhorus-chat-ui-design.md](2026-07-07-qhorus-chat-ui-design.md)
**Scope:** 11 chat platforms, 6 agent communication frameworks, speech act
theory, design dimension analysis

---

## 1. Platform Structural Comparison

### IRC — The Ur-Model

**Hierarchy:** Network → Server → Channel → User

The deliberately structureless model. Channels are ephemeral multicast
groups — created when the first client joins, cease to exist when the last
leaves. No persistence, no threading, no history. Channel prefixes encode
scope: `#` (network-wide), `&` (server-local), `+` (modeless), `!`
(collision-resistant).

Every subsequent platform is a response to what IRC left out. IRC
established: channels as lightweight message routing targets that come and
go organically.

### Slack — Channel-First

**Hierarchy:** Workspace → Channel (flat)

"Everything is a channel." The Conversations API treats all types (public,
private, DM, group DM) through a single interface with 26+ boolean fields
differentiating them.

**Threading:** Timestamp-based reply chains. A message becomes a thread
parent when replies reference its `ts` as their `thread_ts`. Threads are
message collections, not first-class objects — they have no independent
identity. `reply_broadcast: true` optionally surfaces a reply back in the
main channel.

**Structural insight:** Slack Connect (cross-org channels) broke the
"workspace = customer partition" assumption, forcing ID instability and
deduplication challenges.

### Discord — Community-Scale

**Hierarchy:** Guild → Category → Channel → Thread

Three-level nested hierarchy with 19 channel types, all represented as the
same object differentiated by integer `type`.

**Threads are first-class channel objects** with their own IDs, permissions,
rate limits, and membership tracking. They carry `thread_metadata` (archived,
auto_archive_duration, locked, invitable).

**Forum channels** (type 15) invert the model: the channel holds no messages,
only tagged/sorted threads. Each forum post is a titled thread sent
atomically. This bridges chat and forum — structured enough for knowledge
retention, lightweight enough for casual use.

**Structural insight:** Categories are themselves channel objects (type 4),
not a separate construct. Voice and text converge — voice channels
simultaneously function as text channels.

### Microsoft Teams — Dual-Model

**Hierarchy:** Team → Channel → Thread AND Chat (separate system)

Two completely separate conversation systems with different storage and
behavior. Channels store in team-level mailbox (SharePoint for files).
Chats store in personal Exchange mailboxes (OneDrive for files). Reply
mechanisms differ between them (`replyToId` in channels, message
attachments with `contentType: "messageReference"` in chats).

**Structural insight:** The dual model creates inconsistency but reflects
the reality that team-scoped persistent conversations and personal
ephemeral chats have fundamentally different storage and privacy needs.

### Matrix/Element — Federated DAG

**Hierarchy:** Space → Room (everything is a room)

Rooms are the single universal primitive. DMs are rooms with two members
and `is_direct: true`. Spaces (MSC1772/MSC2946) are rooms with
`type: m.space` that define parent-child relationships via state events,
forming recursive hierarchies.

Room history is a directed acyclic graph — each event references parent
events via `prev_events`. State resolution is transitive and
server-state-independent, enabling federation.

**Threading** (MSC3440): `m.thread` relations point to thread root (flat,
not a reply chain). No nested threads.

**Structural insight:** By making everything a room, Matrix achieves
maximum composability at the cost of requiring explicit metadata to
distinguish DMs from group chats from spaces. The recursive space model
is the most flexible hierarchy in the design space.

### Zulip — Topic-Mandatory

**Hierarchy:** Organization → Channel → Topic

Every message must have a topic. This is the model's most distinctive
constraint and its core philosophical position.

**Topics differ from threads:**
- Named (brief but specific: "question about topics", "issue #1234")
- Persistent and long-lived (conversations span hours or days)
- Mandatory (no message exists outside a topic)
- Retrospectively organizable (rename, move, merge, resolve, delete)
- The unit of catch-up — read one topic at a time, in full context

DMs exist outside the channel/topic hierarchy — they have no topics. For
frequent group conversations, Zulip recommends a private channel to gain
topical organization.

**Structural insight:** Making organization a system requirement rather
than a social convention solves the coordination problem. When threading
is optional, people don't use it. The Zulip community regularly runs 20+
simultaneous topics per channel without chaos.

### Google Chat — Evolving Model

**Hierarchy:** Space → Message → Thread

Three threading modes per space: THREADED_MESSAGES (replies in context),
GROUPED_MESSAGES (topic-grouped), UNTHREADED_MESSAGES (flat stream).

**Thread keys** are app-scoped: multiple Chat apps using the same thread
key create different threads. To reply in a thread started by someone
else, you must use the system-assigned thread `name`.

**Structural insight:** App-scoped thread keys acknowledge multi-app
environments — a design consideration relevant for multi-agent systems.

### Mattermost — Open-Source Mirror

**Hierarchy:** Team → Channel (Slack model with modifications)

Collapsed Reply Threads (CRT) architecture uses explicit Threads and
ThreadMemberships tables. Unread state tracked at both channel and thread
level. Deployment with CRT required schema changes shipped months before
feature launch.

**Directional conversions:** Public → Private (one-way), Group Messages →
Private Channels (from v9.1). No reverse — security-forward design.

### Telegram — Migration-Based

**Hierarchy:** Flat (chats exist independently)

Everything beyond basic groups shares the single `channel` constructor,
differentiated by flag bits: `megagroup=true` (supergroup), `broadcast=true`
(channel), `forum=true` (forum topics).

**Automatic one-way migration:** Basic groups upgrade to supergroups
irreversibly at 200 members. The migration creates two objects with a
`migrated_to`/`migrated_from_chat_id` pointer. Chat history splits across
both IDs.

**Forum topics** (2022): Supergroups with the `forum` flag. Each topic
identified by `message_thread_id`, splitting a single group into parallel
conversation streams.

### Twist (by Doist) — Thread-First

**Hierarchy:** Channel → Thread → Comment

Threads are the primary conversation unit, not channels. You don't post
into a channel stream — you create titled threads within channels. DMs are
deliberately lightweight ("not meant to hold important work conversations").

**Inbox-zero workflow:** The inbox shows new content in followed threads.
Users read, comment, mark threads as done. A task-like processing model,
not a monitoring model.

**Structural insight:** Thread-first design is explicitly async-first.
Twist is "weaker when a team expects chat to replace meetings, quick
standups, or rapid-fire problem solving." The trade-off between structure
and spontaneity is real.

### Discourse — Forum-as-Communication

**Hierarchy:** Category → Subcategory → Topic → Post (with orthogonal Tags)

**Flat threading with navigational annotations.** Chronology determines
layout (posts in time order, not reply hierarchy). Reply-to is optional
metadata — a navigational annotation, not a structural organizer. Forward
and backward links let users navigate reply relationships without
restructuring the timeline.

**Trust levels** gate structural authority: TL3 (Regular) can recategorize
and rename topics. TL4 (Leader) can edit all posts, pin/unpin/close/
archive, split and merge topics. Structural power earned, not granted.

**Discourse Chat** (2022): Real-time chat as a parallel mode with
first-class conversion to forum topics for long-term preservation.

**Structural insight:** The platform was "initially designed around a very
flat structure using tags" because "tree structure categories can become a
nightmare and put off new/casual users."

---

## 2. Design Dimensions

### Dimension 1: Threading Philosophy

| Platform | Model | Mandatory? | First-class objects? |
|----------|-------|:----------:|:--------------------:|
| IRC | None | N/A | N/A |
| Slack | Timestamp reply chains | Optional | No (message collections) |
| Discord | Channel-typed thread objects | Optional | Yes (own IDs, permissions) |
| Teams | Every channel message starts a thread | Mandatory in channels | No (reply chains) |
| Matrix | m.thread relations to root | Optional | No (relation metadata) |
| Zulip | Named persistent topics | **Mandatory** | **Yes** (named, movable) |
| Google Chat | Space-level threading mode | Configurable | Partial |
| Mattermost | Collapsed reply threads | Optional | Partial (DB tables) |
| Telegram | Forum topics in supergroups | Optional (forum mode) | Yes |
| Twist | Titled threads in channels | **Mandatory** | **Yes** |
| Discourse | Sequential posts with reply-to | N/A (chronological) | Topics are the unit |

**Finding:** Optional threading degrades to no threading under real usage.
Zulip and Twist prove that mandatory structure works when the system is
designed for it.

### Dimension 2: Hierarchy Depth

| Depth | Platforms |
|-------|----------|
| 1 level (flat) | IRC, Telegram |
| 2 levels | Slack, Mattermost, Google Chat |
| 3 levels | Discord, Zulip, Teams, Discourse |
| Recursive | Matrix |

**Finding:** 3 levels (container → channel → sub-conversation) is the
sweet spot. Recursive nesting (Matrix) provides flexibility but 2-3 levels
covers all real-world patterns.

### Dimension 3: Persistence Model

| Model | Platforms |
|-------|----------|
| No persistence | IRC |
| Configurable retention | Discourse Chat (90d default), Slack |
| Full persistence | All others |

**Finding:** People self-censor more when they believe a record is being
kept. Removing permanence allows more candid exchange but destroys
organizational knowledge. For agent communication, full persistence is
non-negotiable — the audit trail is the value.

### Dimension 4: DM Treatment

| Approach | Platforms |
|----------|----------|
| DM = just another room/channel | Matrix, Slack, Google Chat |
| DM = separate system | Teams, Mattermost |
| DM = deliberately lightweight | Twist |
| DM = outside topic hierarchy | Zulip |

**Finding:** Treating DMs as "just another channel" simplifies the model.
Zulip's choice to exclude topics from DMs acknowledges that 1:1
conversations don't need organizational structure.

### Dimension 5: Focused/Temporary Conversations

| Platform | Mechanism |
|----------|-----------|
| IRC | Channels vanish when empty |
| Slack | Channel archiving, canvases |
| Discord | Thread auto-archiving (60min–7 days) |
| Teams | No explicit mechanism |
| Matrix | Room archiving, tombstone events |
| Zulip | Topic resolving (mark complete) |
| Google Chat | Space deletion |
| Mattermost | Channel archiving (read-only) |
| Telegram | Forum topic close/hide/pin |
| Twist | Thread done/close workflow |
| Discourse | Topic closing, archiving, unlisting |

**Finding:** Resolution (marking something as done/resolved) is more useful
than deletion for agent communication — the conversation history has value
even after the work is complete.

---

## 3. The Central Design Tension

### Structure vs. Spontaneity

**Channel-first** (Slack, IRC, Mattermost): Optimizes for spontaneity and
real-time flow. Low friction. But interleaved conversations create
cognitive load — studies show 65% of messages in flat channels are
off-topic relative to any given reader's context.

**Thread-first** (Twist, Zulip): Optimizes for comprehension and async
participation. Every conversation has a named container. But structure adds
friction — Twist "is weaker when a team expects chat to replace meetings,
quick standups, or rapid-fire problem solving."

**The optional-threading compromise** (Slack, Discord, Mattermost): Offers
threads but doesn't require them. Zulip's critique: "when threading is
optional, people don't use it." The coordination problem means optional
structure degrades to no structure.

### The Convergence Pattern

Platforms are converging toward hybrid models:
- Discourse added real-time chat (2022) alongside forum topics
- Discord added forum channels (2022) alongside real-time chat
- Telegram added forum topics to supergroups (2022)
- Rocket.Chat explicitly implemented both threads and discussions

The consensus: no single model handles all communication needs.

### Implication for Agent Communication

Agent conversations are high-volume, highly parallel, and structured by
nature (speech acts, obligations, correlation chains). The flat stream
model is the worst fit. Zulip's mandatory topic model is the strongest
structural match:

- An orchestrator issues COMMAND "Analyze auth code" → named topic
- STATUS and DONE messages belong to the topic by name
- Humans scan topic names to find relevant conversations
- Topics resolve when work completes
- Multiple simultaneous topics are normal and expected

---

## 4. Agent Communication Frameworks

### FIPA ACL (IEEE Standard, 2002)

22 communicative acts grounded in speech act theory:

| Category | Performatives |
|----------|--------------|
| Information | inform, inform-if, inform-ref, confirm, disconfirm |
| Querying | query-if, query-ref, subscribe |
| Negotiation | cfp, propose, accept-proposal, reject-proposal |
| Actions | request, request-when, request-whenever, agree, refuse, cancel |
| Errors | failure, not-understood |
| Distribution | propagate, proxy |

All performatives reduce to two primitives: `inform` and `request`.

Message structure: 12 fields (only performative required). Includes
conversation-id, reply-with, reply-by — threading built into the protocol.

Interaction protocols: REQUEST, QUERY, Contract Net, English/Dutch
auctions, brokering. Each is a reusable finite state machine.

**Mapping to qhorus:** Qhorus's 9 speech acts are a focused subset of
FIPA's 22. The reduction is appropriate — qhorus drops expressives (no
agent apologies), distribution (handled by channel infrastructure), and
most negotiation (replaced by commitment model).

### KQML (DARPA, 1990s)

FIPA ACL's predecessor. Introduced the performative concept and
communication facilitators (agents that broker/route messages by content).
Established content language independence — protocol envelope separate
from payload.

The format LLMs naturally use for structured communication mirrors KQML's
structure, showing lasting influence.

### Google A2A Protocol (2025, Linux Foundation)

JSON-RPC 2.0 over HTTPS. Agent Cards for capability discovery at
`/.well-known/agent-card.json`.

Task lifecycle: submitted → working → input-required → completed/failed/
canceled/rejected.

Messages (communication) vs Artifacts (outputs) — explicitly separated.
contextId for conversation scoping, taskId for task continuity.

150+ supporting organizations as of 2026.

**Mapping to qhorus:** A2A's contextId ≈ qhorus channelId. A2A's taskId ≈
qhorus correlationId. A2A's task states map to qhorus commitment states.
The models are structurally aligned.

### AG2 (formerly AutoGen)

ConversableAgent as universal base. Group Chat: all agents contribute to a
single shared thread, GroupChatManager selects next speaker. Six
speaker-selection modes including LLM-based.

Nested Conversations: an agent can internally run a multi-agent
conversation while appearing as a single entity externally.

v0.9 (April 2025) adopted an actor model for orchestration, enabling both
point-to-point and broadcast messaging.

### CrewAI

Role-based metaphor with hub-and-spoke communication — no peer-to-peer
agent traffic. Three process types: sequential, hierarchical (manager
delegates), consensual (agents vote).

Delegation via two tools: "Delegate work to coworker" and "Ask question to
coworker." Schema-validated message envelopes.

### LangGraph

Agent coordination as explicit state machines. Three primitives: State
(shared data), Nodes (agent logic), Edges (transitions). Reducer-driven
state schemas prevent data loss in multi-agent updates. Checkpointing
after every node for fault tolerance.

### OpenAI Swarm / Agents SDK

Two primitives: Routines (instructions + tools) and Handoffs (swap active
agent with full history transfer). Stateless. Superseded by Agents SDK
with guardrails, tracing, and human-in-the-loop.

### Anthropic Multi-Agent Patterns

Five workflow patterns: prompt chaining, routing, parallelization,
orchestrator-workers, evaluator-optimizer. One agent pattern: autonomous
agent in a tool-use loop.

Five coordination patterns (April 2026): generator-verifier,
orchestrator-subagent, agent teams, message bus, shared state.

---

## 5. Speech Act Theory Foundations

### Austin/Searle Framework

Austin (1955): three levels — locutionary (the utterance), illocutionary
(the intended action), perlocutionary (the achieved effect).

Searle (1969): five categories of illocutionary acts:
- **Assertives** — describe how things are (inform, confirm)
- **Directives** — get the hearer to do something (request, query)
- **Commissives** — commit the speaker to future action (propose, agree)
- **Expressives** — express psychological states (apologize, thank)
- **Declaratives** — change reality by utterance (declare, resign)

### Mapping to Qhorus

| Searle category | FIPA ACL | Qhorus |
|----------------|----------|--------|
| Assertives | inform, confirm | RESPONSE, STATUS, EVENT |
| Directives | request, query | COMMAND, QUERY |
| Commissives | agree, propose | (implicit in accepting a COMMAND) |
| Expressives | — | (absent — correct for agents) |
| Declaratives | cancel | DONE, FAILURE, DECLINE, HANDOFF |

Modern LLM agent systems implicitly use a reduced speech act vocabulary:
inform, request/delegate, accept/reject, escalate. Qhorus's 9 types sit
between the full FIPA set and the minimal modern set — rich enough for
structured communication, lean enough for practical use.

### Social Commitments

Classical MAS research developed social commitment semantics as an
alternative to mentalistic (BDI) approaches. A social commitment is an
agreement between debtor and creditor agents — publicly observable (unlike
beliefs/desires/intentions), making it formally verifiable.

Qhorus's CommitmentStore implements this directly: COMMAND creates a
commitment (OPEN), DONE fulfills it (FULFILLED), FAILURE terminates it
(FAILED). The 7-state lifecycle (OPEN, ACKNOWLEDGED, FULFILLED, FAILED,
DECLINED, DELEGATED, EXPIRED) matches the theoretical model.

### Governance Gaps (2026 Finding)

A June 2026 paper identified that none of MCP, A2A, or ACP can express:
membership (admission/removal), deliberation (structured argument
exchange), voting, or dissent. These are governance primitives that FIPA
partially addressed but modern protocols have not incorporated.

Qhorus partially addresses this via:
- Channel membership (allowedWriters, adminInstances → qhorus#328 extends)
- The oversight channel (human governance, approval gates)
- The commitment model (obligation tracking)

The deliberation and voting gaps remain open — relevant if agents need to
reach consensus through structured debate.

---

## 6. Terminology Decisions

### Why "Space" (not "Room" or "Group")

| Term | Existing usage | Problem |
|------|---------------|---------|
| Room | Matrix, Slack huddles | Implies a persistent place you enter/leave — channels already do this |
| Group | Telegram, Teams | Implies an ad-hoc collection of people — that's membership, not structure |
| Space | Matrix | Container above channels — clean, established, no collision with qhorus concepts |

### Why "Topic" (not "Thread" or "Subject")

| Term | Existing usage | Problem |
|------|---------------|---------|
| Thread | Slack, Discord, Mattermost | Anonymous, created by replying, tends to die. Not the right mental model for agent work. |
| Subject | Email | Carries email baggage. Doesn't imply persistence or navigability. |
| Topic | Zulip, Telegram forums, Discourse | Named, persistent, intentionally created. The unit of catch-up. Proven at scale. |

### Why "Thread" as the sub-topic level (not "Chain" or "Conversation")

Thread is the universal term for a reply chain within a scoped context. In
our model, threads are implicit (formed by correlationId or inReplyTo) —
not created explicitly. "Chain" is too technical. "Conversation" is too
broad (the whole channel is a conversation).

---

## 7. Research for AI-Native Communication

### Key Findings from Multi-Agent Research

- 40.5% of users cannot find information through conversational interfaces;
  50.9% fail to reach goals due to misaligned chat flows
- 95% of contemporary AI tools operate statelessly — each query in isolation
- Chat history is "underdesigned in most AI products" — users want to share,
  reference, and continue past conversations
- Memory-augmented approaches reduce token usage by 90%+ while maintaining
  accuracy
- Agent heterogeneity (different foundation models) is a dominant driver
  of debate success — homogeneous debates converge prematurely
- Most debate frameworks do not consistently outperform simpler single-agent
  strategies — structured debate is not universally better
- Risk of echo chambers and eloquent-but-wrong persuasion in multi-agent
  debate

### Dual-Channel Finding (July 2026)

"What LLM Agents Say When No One Is Watching" introduced a dual-channel
framework where agents produce both public utterances and private
"off-the-record" responses. Social dynamics undermine objective
decision-making in LLM collectives.

**Mapping to qhorus:** The work/observe/oversight triple already captures
this: work channel for public obligation exchange, observe channel for
private telemetry (not delivered to agent context), oversight channel for
human governance. The three-channel separation was independently validated
by this research.

### Human-in-the-Loop UI Research

- Magentic-UI (Microsoft, 2025): co-planning, co-tasking, action approval,
  final answer verification. 74.58 SUS usability score.
- AG-UI protocol: real-time streaming of agent reasoning, tool approval
  interrupts
- Agentic design patterns: progressive disclosure, confidence visualization,
  mixed-initiative controls

### Structural Implication

AI agents need conversation models closer to Zulip's topic model (named,
persistent, resumable) than IRC's stream model (stateless, ephemeral).
Thread identity and conversation history are prerequisites, not optional
features.

---

## Sources

### Chat Platforms
- Slack: [Conversations API](https://docs.slack.dev/apis/web-api/using-the-conversations-api/), [Conversation object](https://docs.slack.dev/reference/objects/conversation-object/)
- Discord: [Channel resource](https://discord.com/developers/docs/resources/channel)
- Teams: [Graph API overview](https://learn.microsoft.com/en-us/graph/api/resources/teams-api-overview)
- Matrix: [MSC3440 Threading](https://github.com/matrix-org/matrix-spec-proposals/blob/main/proposals/3440-threading-via-relations.md), [MSC2946 Spaces](https://github.com/matrix-org/matrix-spec-proposals/blob/main/proposals/2946-spaces-summary.md)
- Zulip: [Why Zulip](https://zulip.com/why-zulip/), [Intro to topics](https://zulip.com/help/introduction-to-topics)
- Google Chat: [Spaces API](https://developers.google.com/workspace/chat/api/reference/rest/v1/spaces)
- Mattermost: [Scaling CRT](https://mattermost.com/blog/scaling-collapsed-reply-threads/)
- Telegram: [Bot API](https://core.telegram.org/bots/api), [MTProto Channels](https://core.telegram.org/api/channel)
- Twist: [Doist design blog](https://doist.com/blog/designing-twist-the-challenge-of-making-teamwork-less-stressful/)
- Discourse: [Trust levels](https://blog.discourse.org/2018/06/understanding-discourse-trust-levels/), [Chat intro](https://meta.discourse.org/t/introducing-discourse-chat-beta/210734)
- IRC: [RFC 2812](https://datatracker.ietf.org/doc/html/rfc2812)

### Agent Communication
- FIPA ACL: [SmythOS overview](https://smythos.com/developers/agent-development/fipa-agent-communication-language/), [Performatives](https://jmvidal.cse.sc.edu/talks/agentcommunication/performatives.html)
- A2A: [Specification](https://a2a-protocol.org/latest/specification/), [Google announcement](https://developers.googleblog.com/en/a2a-a-new-era-of-agent-interoperability/)
- AG2: [v0.9 release](https://docs.ag2.ai/latest/docs/blog/2025/04/28/0.9-Release-Announcement/)
- CrewAI: [Collaboration docs](https://docs.crewai.com/en/concepts/collaboration)
- LangGraph: [Framework guide](https://latenode.com/blog/ai-frameworks-technical-infrastructure/langgraph-multi-agent-orchestration/)
- Swarm: [GitHub](https://github.com/openai/swarm)
- Anthropic: [Building Effective Agents](https://www.anthropic.com/research/building-effective-agents), [Multi-Agent Coordination](https://claude.com/blog/multi-agent-coordination-patterns)

### Research
- [Threading impact on social reciprocity](https://www.researchgate.net/publication/325999964)
- [Dialog history and task performance](https://www.academia.edu/256627)
- [LLM agents when no one is watching](https://arxiv.org/html/2607.02507)
- [Multi-agent communication patterns](https://www.mdpi.com/2076-3417/15/13/7267)
- [Singh: Commitments in MAS](https://www.csc2.ncsu.edu/faculty/mpsingh/papers/mas/Commitments-for-MAS.pdf)
- [Governance gaps in MCP/A2A/ACP](https://arxiv.org/html/2606.31498v1)
- [Magentic-UI](https://www.microsoft.com/en-us/research/wp-content/uploads/2025/07/magentic-ui-report.pdf)
- [Cost of ephemeral communication](https://blog.discourse.org/2025/10/the-cost-of-ephemeral-communication/)
