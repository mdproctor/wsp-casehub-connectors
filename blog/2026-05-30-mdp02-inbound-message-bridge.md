---
title: "Inbound Goes Both Ways"
date: 2026-05-30
tags: [casehub-connectors, casehub-qhorus, inbound, HumanParticipatingChannelBackend, architecture]
---

I knew going into this that connectors#6 was going to be harder than it looked. The issue title —
"InboundMessage CDI event → MessageService.dispatch()" — sounds like a bridge: two things you connect with a cable.
It turned out to be an architecture problem dressed as a plumbing problem.

The starting point was a design session that I'd expected to take an hour and took most of a day. Before
anything compiled, I needed to figure out where the bridge should live, what the channel granularity
should be, and what `externalChannelRef` actually means in practice. That last one is where the real
design insight came from.

I read through all four inbound connectors before proposing anything. What I found: `externalChannelRef`
is always our own endpoint. For Twilio SMS, it's `params.get("To")` — our Twilio number. For WhatsApp,
it's `phone_number_id` — our Meta phone ID. For email, it's the recipient address — our inbox. The sender
is always `externalSenderId`. This matters because if you're going to do per-conversation Qhorus channels,
you need to key on the sender, not the channel ref. The channel ref just says "this arrived at our system"
— it doesn't say "this is a conversation with Alice."

I'd initially planned three options — a CDI event decoupling approach (Approach B), the full
`HumanParticipatingChannelBackend` integration (Approach C), and a simple runtime dependency approach (A).
After reading the ChannelGateway code and all the existing backend implementations (Claudony, OpenClaw),
I recommended B and the user picked C. C is architecturally cleaner because it wires into the existing
backend lifecycle: `ChannelInitialisedEvent` for registration, `post()` for outbound, and a proper binding
in the channel entity rather than external configuration.

The brainstorming had a code review baked in by design. I wrote the spec, got a thorough review back
with a critical finding (the original spec had `@ObservesAsync` receiving from a synchronous `Event.fire()`
— that's a CDI 2.0 no-op; the spec was wrong), and revised it twice before touching any code. The review
also surfaced the `ChannelConnectorBinding` table design — I'd initially added four nullable columns
to the `Channel` entity, which is the standard fat-entity smell. A separate binding table with `channelId`
as PK makes the "all four or none" invariant structural, not application-enforced. That's the right call
and I wish I'd seen it first.

---

The implementation was 18 tasks across two repos. I brought Claude in as the implementer and ran
code reviews after every task. The structure was strict: spec compliance first, code quality second,
no skipping.

The `InboundConnectorService` change came first: swap `messageEvent::fire` for a `fireAsync()` lambda
with an `.exceptionally()` handler. Simple change, significant contract break — every existing
`@Observes InboundMessage` consumer would now get nothing. There were zero production consumers, so
the blast radius was clean, but we still had to update both test capture beans from `@Observes` to
`@ObservesAsync` with `BlockingQueue`-based patterns to match. The tests confirm the TDD intention:
set them up to fail first (ICS still sync, capture uses async observer), then fix ICS, watch them pass.

The qhorus data layer went smoothly until it didn't. `ChannelBindingStore` follows the store seam
protocol — interface in `runtime/store`, JPA impl in `runtime/store/jpa`, in-memory in `testing/`.
`ChannelCreateRequest` is a compact-constructor record that enforces the binding invariant. `ChannelDetail`
gained a `ConnectorBinding` nested record as the 14th field — a binary break on the canonical constructor,
which required updating `QhorusEntityMapper` and `QhorusDashboardService`. The review caught that both
had independent implementations of the binding lookup, which would diverge silently. The fix was to
have the dashboard delegate to the mapper instead of maintaining its own copy.

The `ConnectorChannelBackend` itself is one `@ApplicationScoped` bean with an in-memory
`ConcurrentHashMap` cache keyed on channel UUID. Registration happens via `@Observes ChannelInitialisedEvent`
— it checks the binding store, populates the cache, then deregisters-and-registers with the gateway
(idempotent, same pattern as `ClaudonyChannelBackend`). Inbound routing uses `@ObservesAsync InboundMessage`,
derives the lookup key via `ConnectorKeyStrategy` (externalSenderId for SMS/WhatsApp/Email, externalChannelRef
for Slack), and delegates to `gateway.receiveHumanMessage()`. Outbound delivery reads from the cache
in `post()` — no database access during fanOut, which is important because fanOut runs in a virtual thread.

`ConnectorKeyStrategy` is the architectural heart. The table it encodes:

| Connector | Lookup key | Why |
|-----------|-----------|-----|
| slack-inbound | externalChannelRef | Slack channel IS the conversation |
| twilio-sms-inbound | externalSenderId | externalChannelRef is our number |
| whatsapp-inbound | externalSenderId | externalChannelRef is our phone ID |
| email-inbound | externalSenderId | externalChannelRef is our inbox |

No framework magic. Just a `Set.of(...)` with the sender-keyed connectors and a default fallback.

---

The integration test took the most debugging time, and for reasons that had nothing to do with the code.

Running `mvn test -pl connector-backend` fails with `casehub-ledger-deployment-0.2-SNAPSHOT.jar does not exist`
because Quarkus `@QuarkusTest` needs the deployment JARs of every extension on the classpath for augmentation,
and those jars only exist after `mvn install` — not after `compile` or `test-compile`. This is a multi-repo
problem: if `casehub-ledger` hasn't been installed to `~/.m2` (with its deployment module), the test can't
augment. The fix is to `mvn install -DskipTests` the dependency chain before running isolated tests.

There was also the `*.lastUpdated` issue. After a GitHub Packages 401, Maven writes failure markers to
`~/.m2` that persist past the session. Fixing auth doesn't help — Maven reads the marker and refuses to
retry. You have to delete the `.lastUpdated` files manually (or via Python pathlib) before the next
resolution attempt will succeed. I lost a couple of hours to this before finding the pattern.

The connector-backend pom also had a wrong artifactId: I'd written `casehub-qhorus-runtime` but the
runtime module's actual Maven artifactId is `casehub-qhorus`. That's a naming choice from before the
module structure was fully settled — the folder is `runtime/` but the artifact is the parent name.
The test couldn't find the JAR, Maven fell through to GitHub Packages, 401, cached failure, confusion.
Once the artifactId was fixed the dependency graph resolved correctly.

---

The final code review found two real issues I'd missed. `ChannelCreateRequest`'s compact constructor
checked `inboundConnectorId != null → all four required` but not the reverse: if you passed
`externalKey = "something"` with the others null, it silently accepted it. The invariant comment said
"all four or none" but the code said something weaker. Fixed to check `anySet && !allSet` before throwing.

The second was `QhorusDashboardService` maintaining its own `toChannelDetail()` method independently
of `QhorusEntityMapper` — both building `ChannelDetail.ConnectorBinding` with their own binding store
lookups. A one-line delegate call fixes the duplication. The kind of thing that looks fine in isolation
and becomes a maintenance sink when the binding mapping needs to change.

Both fixed, committed, done.

---

What ships: `ConnectorChannelBackend` in `casehub-qhorus`, wired to `InboundMessage` from
`casehub-connectors`, per-conversation channel granularity, pre-provisioned bindings only in v1.
Six deferred issues filed: auto-channel creation, cache refresh on binding update, per-connector
normaliser, MCP tools, overload consolidation, and the magic string constants in `ConnectorKeyStrategy`.

The most interesting architectural finding from this whole session: `externalChannelRef` has been
carrying the wrong semantic from the start. It identifies our endpoint — the Twilio number we own,
the inbox we receive at — not the conversation partner. That's fine for Slack where the channel
is genuinely the conversation space, but it means the bridge has to know, per connector type, which
field actually identifies who you're talking to. That asymmetry is now encoded in `ConnectorKeyStrategy`
and will need to stay there until someone rethinks what `externalChannelRef` means at the source.
