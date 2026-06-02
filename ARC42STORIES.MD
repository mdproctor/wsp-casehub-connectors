# casehub-connectors — ARC42STORIES.MD

**Spec:** Arc42Stories v0.1
**Profile:** CaseHub — Foundation tier
**Profile ref:** `../parent/docs/arc42stories-casehub-profile.md` · fallback: `https://raw.githubusercontent.com/casehubio/parent/main/docs/arc42stories-casehub-profile.md`
**Build position:** After casehub-platform; no other casehubio deps. Publishes before application-tier apps.
**Consumed by:** Application-tier harness apps that need outbound/inbound message delivery (Slack, Teams, SMS, email, webhooks)
**Depends on:** None (or casehub-platform-api for shared types). Zero casehubio foundation deps beyond platform.

---

## §1 Introduction and Goals

### Description

`casehub-connectors` provides outbound and inbound message connectors — the delivery layer for notifications, escalations, and webhook integrations. It decouples application-tier domain logic from transport-specific implementation details.

Core capabilities: outbound connectors (Slack, Teams, SMS, email), inbound connectors (webhook receiver, IMAP email inbound), connector SPI for application-tier extension, message routing and retry semantics.

The connector layer handles the mechanics of delivery; application-tier domain logic dispatches to connectors via the SPI without knowing the transport. This separation means a new transport (e.g. WhatsApp) can be added without touching any application code.

### Stakeholders

| Stakeholder | Interest |
|---|---|
| Application-tier harness apps | Consume this module's foundation primitives |
| Platform team | Maintain correctness and backward compatibility |
| LLM sessions (Claude) | Navigate architecture and implement extensions confidently |

### Quality Goals

| Goal | Scenario |
|---|---|
| Transport isolation | Application-tier domain logic never references transport-specific types — all delivery goes through the connector SPI |
| Retry semantics | Outbound connectors handle transient failures without burdening application logic with retry state |

---

## §2 Constraints

### Platform
Java 21 (on Java 26 JVM), Quarkus 3.32.2. Published to GitHub Packages as `io.casehub:casehub-connectors:0.2-SNAPSHOT`.

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn clean install
```

### Dependencies
Resolved from GitHub Packages: `https://maven.pkg.github.com/casehubio/*`. CI authentication via `GITHUB_TOKEN`.

---

## §3 Context and Scope

🔲 C4Context diagram — add in dedicated session.

**Boundary rules:** casehub-connectors owns only the concerns described in §1. Domain logic (business rules, application-specific workflows) belongs in application-tier repos (devtown, aml, clinical, life, drafthouse, quarkmind).

See `../parent/docs/PLATFORM.md` Capability Ownership table for the full boundary map.

---

## §4 Solution Strategy

🔲 Core architectural patterns and module structure — complete in dedicated session from DESIGN.md and repo architecture.

---

## §5 Building Block View

🔲 C4Container diagram and module structure — complete in dedicated session.

---

## §6 Runtime View

🔲 Key runtime scenarios — complete in dedicated session.

---

## §7 Deployment View

Published to GitHub Packages. CI: GitHub Actions. No GraalVM native target (foundation modules).

---

## §8 Crosscutting Concepts

References to governing protocols:

| Concern | Protocol / Reference |
|---|---|
| Module structure | `docs/protocols/universal/module-tier-structure.md` |
| Flyway migrations | `docs/protocols/universal/flyway-migration-rules.md` |
| CDI displacement (`@DefaultBean`) | `docs/protocols/casehub/alternative-extension-patterns.md` |
| Capability ownership | `../parent/docs/PLATFORM.md` Capability Ownership table |
| Architectural patterns | `../parent/docs/ARCHITECTURE.md` |

🔲 Anti-patterns inline (Symptom → Cause → Fix) — complete in dedicated session. Must be present per profile rules; do not defer to AGENTIC-HARNESS-GUIDE.md reference alone.

---

## §9 Journeys and Chapters

🔲 **Bootstrap — full Journeys and Chapters pending dedicated session.**

Foundation modules use Chapters to track delivery milestones of the foundation capability itself (transport types, SPI increments, new adapters) rather than integrating external foundation modules.

Read `../parent/docs/arc42stories-spec.md §4 Solution Strategy` — "Foundation: Chapters = delivery milestones of the foundation capability itself."

The full §9 entry requires:
- Reading DESIGN.md (or equivalent) for the delivery arc
- Mapping milestones to Chapters
- Writing §9.4 Layer/Chapter entries with all 18 required subsections

---

## §10 Architectural Decisions

Per the Arc42Stories spec: only decisions not captured inline elsewhere. Should be sparse.

🔲 Cross-cutting decisions — complete in dedicated session.

---

## §11 Quality Requirements

See §1 Quality Goals for the primary scenarios.

🔲 Full quality requirements table — complete in dedicated session.

---

## §12 Risks and Technical Debt

🔲 Active risks and technical debt — complete in dedicated session.

---

## §13 Glossary

| Term | Definition |
|---|---|
| `Connector SPI` | Domain-agnostic dispatch interface — application tier calls the SPI; the connector handles transport specifics. |
| `Outbound connector` | Delivers messages to external systems (Slack, Teams, SMS, email). |
| `Inbound connector` | Receives messages from external systems (webhooks, IMAP email) and routes them into the case context. |

---

*Bootstrap document — generated 2026-06-02. §3–§9 require a dedicated session to complete from repository source material.*
