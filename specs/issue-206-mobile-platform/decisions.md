# MobilePlatform SPI — Design Decisions

## D1: Voice call abstraction model

**Choice:** Instruction-based at the SPI surface — agent provides a goal/instruction as text, receives transcript and outcome as text. No audio-level API (speak/listen with raw bytes) in v1.
**Alternatives:**
- Audio-streaming — agent manages TTS/STT turn-by-turn via bidirectional audio channel. More powerful but impractical for MCP tools and playbooks; deferred to v2 as a separate VoiceCallStreaming capability
- Both from day one — instruction + streaming capabilities. Larger scope without clear v1 need
**Rationale:** Dial's API is natively instruction-based (outboundInstruction). The agent sets a goal ("book a table for 7pm, flexible within 1 hour"), the provider's AI converses freely with the human until the goal is achieved or the call ends. Nothing is scripted. This maps cleanly to MCP tools and playbook steps. The design brief described an audio-streaming model (speak/listen with audio bytes); this decision explicitly replaces that model with text-level interaction. D8's platform-managed mode extends this to interactive text-level conversation with mid-call tools — still text at the SPI surface, even though provider implementations may handle audio internally.
**Trade-offs:** Cannot do real-time agent-controlled AUDIO conversation in v1 — no speak(audio)/listen() at the SPI surface. Text-level agentic conversation with mid-call tools IS in scope via D8's platform-managed mode. Agents that need direct audio control must wait for a VoiceCallStreaming capability in v2.
**Sources:** Dial API docs (docs.getdial.ai), design-brief.md voice architecture section, TravelPlatform SPI pattern
**Exploration:** quick
**Status:** revised — clarified text-vs-audio distinction; updated trade-offs to accurately reflect D8 scope (from R1-02)

## D2: Capability scope for v1

**Choice:** All five capabilities — Sms, VoiceCall, VoiceCallControl, Voicemail, Messaging
**Alternatives:**
- Sms + VoiceCall + Messaging only — smaller surface, but VoiceCallControl and Voicemail are straightforward and Dial supports the underlying operations
- Sms + VoiceCall only — minimal scope, leaves Dial's messaging and call control unused
**Rationale:** Dial natively supports all five: SMS, instruction-based voice calls, call stop/transfer, and WhatsApp/iMessage messaging. Voicemail is list/get/transcribe over provider data. VoiceCallControl (stop, transfer) maps to Dial's call control endpoints and is orthogonal to the instruction-based conversation model.
**Trade-offs:** Larger initial implementation surface. Mitigated by Dial covering all capabilities in one provider.
**Sources:** Dial API docs — POST /calls/:id/stop, transferTo parameter, POST /messages with channel selection
**Exploration:** quick
**Status:** captured

## D3: mobile-ref design

**Choice:** Standard in-memory reference implementation with stateful switchboard — follows chat-ref/bank-ref/calendar-ref pattern. Canned simulation scenarios handled by @SimulationEligible (ADR-0011).
**Alternatives:**
- Dual-mode (canned scripts + LLM-driven by profile) — two code paths in the ref module
- Canned-only — simpler but no path to realistic testing
- LLM-only — richer but can't run in CI without an LLM provider
- Pluggable handler SPI — flexible but over-engineered for a ref module
**Rationale:** mobile-ref is a stateful reference implementation providing in-memory switchboard routing for agent-to-agent voice calls, SMS log, and voicemail storage. This follows the established ref pattern used by chat-ref (ChatBackend), bank-ref (pre-loaded UK data), calendar-ref (monotonic version tracking), and voice-ref (WAV byte wrapping). Canned simulation scenarios are the simulation framework's responsibility — @SimulationEligible on MobilePlatform (D13) provides strategy selection (sequential, key-lookup, random, replay), corpus seeding, and overlay stack, eliminating hand-written simulation code. ADR-0011 explicitly states: "simulation replaces sim/demo modules, not ref modules." The ref module provides state coherence; simulation provides canned scenarios.
**Trade-offs:** No built-in LLM mode in mobile-ref. Dev/demo LLM enrichment can be provided via a profile-gated test fixture or a separate demo module if needed — not baked into the ref architecture.
**Sources:** ADR-0011 simulation framework, chat-ref pattern, bank-ref pattern, voice-ref WAV simulation
**Exploration:** quick
**Status:** revised — dropped dual-mode; ref module follows established pattern, canned simulation via @SimulationEligible (from R1-10)

## D4: Twilio in v1 scope

**Choice:** Include mobile-twilio in v1 alongside mobile-dial
**Alternatives:**
- Defer Twilio to v2 — smaller scope, but delays SPI abstraction validation across providers
- Twilio SMS + messaging only, defer voice — 80% validation at 20% cost
**Rationale:** Having two providers from day one proves the SPI abstraction works across fundamentally different backends. Dial is instruction-based (provider handles conversation), Twilio requires building the conversation loop (Media Streams WebSocket + VoicePlatform STT/TTS + LLM). This validates the SPI is a real abstraction, not a Dial wrapper. Voice is where the backends diverge architecturally — if the SPI handles both Dial's native instruction model AND Twilio's build-from-audio model, it handles anything. Deferring Twilio voice to v2 would defer the hardest validation question to when the SPI is less malleable.
**Trade-offs:** Twilio's voice implementation is substantially more complex — it must build the conversation layer that Dial provides natively (receive audio → STT → LLM → TTS → send audio). This is a multi-week effort. For SMS and messaging, Twilio is straightforward. Phasing within v1: SMS + messaging first, then voice — so the SPI is validated incrementally.
**Depends on:** D1 (instruction-based model — Twilio must implement this on top of raw audio), D12 (VoicePlatform dependency for STT/TTS)
**Sources:** Existing TwilioSmsConnector in codebase, Twilio Media Streams docs
**Exploration:** quick
**Status:** captured

## D5: Legacy connector absorption

**Choice:** Absorb TwilioSmsConnector and WhatsAppInboundConnector into mobile-twilio
**Alternatives:**
- Leave in place as parallel path — less risk but two ways to send SMS, confusing for consumers
- Deprecate without deleting — gradual migration but defers cleanup
**Rationale:** Clean break. One path for all telephony. The old connectors use the fire-and-forget Connector SPI, not the capability-based platform pattern. Moving their logic into mobile-twilio's Sms and Messaging capabilities eliminates duplication and gives consumers a single API.
**Trade-offs:** This is a coordinated cross-module migration, not a simple absorption. TwilioSmsConnector has 25 references across the codebase: CDI producers in ConnectorsBeans (core), Spring auto-config in ConnectorsAutoConfiguration (connectors-spring), notification-bridge module (channelType "sms"), send_sms MCP tool, and documentation. WhatsAppInboundConnector has 19 references with the same pattern. The migration requires: (1) delete old connectors from connectors-api, (2) remove CDI/Spring producers, (3) update or replace MCP tools, (4) preserve notification-bridge channelType contract (mobile-twilio must return "sms" and "whatsapp" for notification delivery compatibility), (5) reimplement all capabilities in mobile-twilio's SPI, (6) update documentation. All references are internal to connectors — no external consumers depend on these classes directly.
**Depends on:** D4 (Twilio in v1)
**Sources:** TwilioSmsConnector.java (25 references), WhatsAppInboundConnector.java (19 references), notification-bridge module, ConnectorsBeans.java, ConnectorsAutoConfiguration.java
**Exploration:** quick
**Status:** revised — acknowledged full migration scope; enumerated cross-module changes required (from R1-06)

## D6: Numbrrs in v1 scope

**Choice:** Defer mobile-numbrrs to v2
**Alternatives:**
- Include in v1 — three providers proves the SPI scales
**Rationale:** Two providers (Dial + Twilio) already validate the SPI across fundamentally different backends (instruction-based vs build-from-audio). A third SMS-only provider adds a module to maintain, tests to write, and credentials to manage without answering any question that isn't already answered. Both Dial and Twilio support SMS — the SMS capability is already validated twice. Numbrrs's UK-only, SMS-only profile does not test any new capability dimension (e.g. a voice-only provider would test something new). Once the SPI is proven with Dial + Twilio, adding Numbrrs later is trivial.
**Trade-offs:** No third provider validation in v1. Mitigated by Dial and Twilio being architecturally different enough to stress-test the SPI.
**Sources:** Numbrrs API (numbrrs.com), issue #206
**Exploration:** quick
**Status:** revised — deferred to v2; two providers sufficient for SPI validation (from R1-08)

## D7: Voicemail transcription timing

**Choice:** Configurable — provider config flag for eager vs lazy transcription
**Alternatives:**
- Lazy only — simpler but no option for pre-transcribed voicemails
- Eager only — wastes resources on unread voicemails
**Rationale:** Different deployments have different needs. A config flag (`casehub.mobile.voicemail.transcribe-on-receive=true|false`, default false/lazy) lets operators choose. Lazy delegates to VoicePlatform STT on access; eager triggers transcription on inbound voicemail event.
**Trade-offs:** Two code paths for voicemail transcription. Mitigated by both paths using the same VoicePlatform STT — the only difference is when. Runtime dependency: voicemail transcription requires a deployed VoicePlatform with a real SpeechToText implementation (voice-sherpa with Whisper, or equivalent). Without one, NoOpVoicePlatform's STT returns empty results — transcription silently produces nothing. This is the graceful degradation pattern (same as ChatPlatform NoOp capabilities), but consumers must be aware at deployment time that voicemail transcription requires VoicePlatform STT.
**Sources:** VoicePlatform STT integration, EmailPlatform lazy attachment content pattern
**Exploration:** quick
**Status:** revised — documented VoicePlatform STT runtime dependency (from R1-09)

## D8: Two call modes — provider-managed and platform-managed

**Choice:** SPI supports both provider-managed (simple instruction) and platform-managed (interactive text-level with mid-call tool access) voice calls
**Alternatives:**
- Platform-managed only — simpler SPI but can't use Dial's simple instruction mode for easy cases
- Provider-managed only — misses the key value of agentic calls (mid-call decisions, human-in-the-loop)
**Rationale:** Simple calls ("book a table for 7pm") can use Dial's built-in AI via `POST /calls`. Complex calls where the agent needs mid-call tool access (check with human via work queue, call another API) use Dial's self-hosted LLM WebSocket — the platform's conversation handler (D9) handles conversation logic with full tool access. Twilio's Media Streams provides the same agentic mode. The SPI abstracts both behind a single call interface — if a ConversationHandler is provided, the provider uses platform-managed mode; otherwise provider-managed. Both modes operate at the TEXT level at the SPI surface (D1). Audio handling (WebSocket, STT/TTS, codec management) is internal to provider implementations — it is never exposed at the SPI surface. For Dial, platform-managed mode is text-over-WebSocket (Dial handles audio). For Twilio, the provider module internally bridges audio to text via VoicePlatform STT/TTS (D12).
**Trade-offs:** Platform-managed mode adds complexity inside provider implementations: Dial requires WebSocket integration with text-level protocol; Twilio requires Media Streams WebSocket + STT/TTS pipeline + conversation loop. This complexity is contained within provider modules — the SPI consumer sees only text. Provider-managed mode is simple REST.
**Depends on:** D1 (text-based SPI surface), D4 (Twilio in v1 — validates both modes work across providers), D9 (ConversationHandler for platform-managed mode)
**Sources:** Dial self-hosted LLM WebSocket protocol (wss://host/path/call_id, HMAC-SHA256 auth, response_required/response messages), Twilio Media Streams WebSocket
**Exploration:** quick
**Status:** revised — clarified that both modes operate at text level at SPI surface; audio handling is internal to provider implementations (from R1-02)

## D9: Platform-managed call conversation handling

**Choice:** Platform-managed calls accept a ConversationHandler callback interface — connectors provides transport, caller provides conversation logic
**Alternatives:**
- AgentDescriptor from eidos-api — connectors depends on eidos-api and launches a proper agent. Violates the "pure delivery infrastructure" boundary: connectors would need to know about agent goals, tools, and disposition. Creates a cross-repo foundation dependency (eidos-api) that doesn't exist today. No other platform SPI launches agents — ChatPlatform doesn't, EmailPlatform doesn't, VoicePlatform doesn't.
- CDI event-driven — looser coupling but complex event flow for real-time conversation
- Built-in conversation loop — breaks the "pure delivery infrastructure" rule
**Rationale:** ARC42STORIES §1 states: "no business logic, routing, scheduling, templating, or retry orchestration. This library delivers messages, fires CDI events, and exposes chat capabilities. Everything else is a caller concern." Agent orchestration — launching an agent with goals, tools, and disposition — is "everything else." The clean architecture: connectors provides WebSocket transport to Dial/Twilio, and the caller provides a ConversationHandler that receives human text and returns response text. The caller (in casehub-engine or casehub-agency, which already depend on eidos-api) wraps an eidos agent in the handler. Connectors doesn't need to know what backs the handler — it could be an agent, a scripted conversation, or a simple echo. This follows the same pattern as WebhookInboundConnector: connectors handles transport, callers handle logic via CDI events.
**Trade-offs:** Caller must wire the ConversationHandler to their agent infrastructure. This is a feature, not a cost — it keeps connectors transport-pure and lets callers use whatever conversation logic they want.
**Depends on:** D8 (platform-managed calls), D1 (text-based SPI surface — handler receives text, not audio)
**Sources:** ARC42STORIES.MD §1 library rule, ChatPlatform pattern (no agent dependency), existing callback patterns in connectors
**Exploration:** quick
**Status:** revised — replaced AgentDescriptor/eidos-api with ConversationHandler callback to preserve boundary rule (from R1-03)

## D10: NumberInfo capability

**Choice:** Include NumberInfo as a read-only capability — listNumbers and lookupCapabilities only. Provisioning (provisionNumber, releaseNumber) deferred to v2.
**Alternatives:**
- Full NumberManagement (list, provision, release, lookup) — includes administrative mutations with billing implications ($3/mo per number on Dial)
- Defer entirely — handle number queries externally
- Separate InfrastructurePlatform SPI — clean separation but over-engineered for read-only queries
**Rationale:** Agents need to know what numbers are available and what each number supports (SMS, voice, WhatsApp) before initiating communication. listNumbers() and lookupCapabilities() serve this need — they answer "what can I use?" without mutating infrastructure. No other platform SPI in connectors includes provisioning operations: BankPlatform doesn't manage bank accounts, CalendarPlatform doesn't manage calendar instances, ChatPlatform's ChannelManagement creates communication resources (channels where messages flow), not infrastructure resources. Number provisioning is a deployment-time infrastructure concern, not a per-call runtime operation. The core use case (dinner booking) uses existing numbers — the agent doesn't provision a new number mid-task.
**Trade-offs:** Agents cannot autonomously provision numbers in v1. Provisioning can be added as a NumberManagement capability in v2 if the use case materialises, or as a separate administrative SPI.
**Sources:** Dial Numbers API (GET /numbers, GET /lookup/number), BankPlatform/ChatPlatform precedent
**Exploration:** quick
**Status:** revised — reduced to read-only queries; provisioning deferred to v2 (from R1-05)

## D11: SPI composition pattern

**Choice:** Composed capabilities with Builder and graceful degradation, following the ChatPlatform pattern
**Alternatives:**
- Flat capabilities (BankPlatform/VoicePlatform pattern) — id(), supports(Class<?>), direct capability accessor methods. No builder. Providers implement all methods; unsupported capabilities throw UnsupportedCapabilityException. Simpler for SPIs with 2 capabilities and homogeneous providers.
- Monolithic interface — single interface with all operations. Forces empty implementations everywhere.
**Rationale:** MobilePlatform has 5+ capabilities and providers with heterogeneous capability profiles (Dial: all five; Twilio: SMS + voice + messaging; a future SMS-only provider: SMS only). The flat BankPlatform pattern (2 capabilities, all providers support both) doesn't scale to this heterogeneity. ChatPlatform faced the same problem (IRC: 3 native, Discord: 8 native, Slack: 10 native) and solved it with composed capabilities + Builder + NoOp degradation defaults. MobilePlatform should follow this proven pattern: Builder assembles from individual capability instances; unsupported capabilities get degradation defaults automatically; supports(Class<?>) reports what's native. Each provider uses the Builder, providing only the capabilities it supports. Callers never null-check — they call the method and get correct behaviour even when unsupported. For critical operations (voice calls, SMS) where silent no-op would be surprising, the NoOp implementations should log a WARNING and return a failure result rather than silently succeeding.
**Trade-offs:** Builder adds ceremony vs flat interface. Justified by the heterogeneous provider landscape.
**Sources:** ChatPlatform.Builder pattern, ChannelFallbackThreading, NoOpReactions, EmptyDiscovery degradation defaults
**Exploration:** quick — surfaced by reviewer as implicit decision
**Status:** captured (from R1-04)

## D12: MobilePlatform / VoicePlatform relationship

**Choice:** Provider modules depend on voice-spi as needed — VoicePlatform is a peer SPI, not a parent or embedded dependency of MobilePlatform
**Alternatives:**
- VoicePlatform as a dependency of mobile-spi — couples the SPI definition to voice infrastructure
- VoicePlatform injected by caller and passed to MobilePlatform — caller must know about internal provider needs
- Each provider brings its own STT/TTS — duplicates voice infrastructure
**Rationale:** MobilePlatform and VoicePlatform operate at different abstraction levels: VoicePlatform is codec-level (audio ↔ text conversion), MobilePlatform is telephony-level (phone calls and messaging). The relationship is a provider implementation detail, not an SPI-level concern. Dial's provider (mobile-dial) does NOT need VoicePlatform — Dial handles audio/TTS/STT internally. Twilio's provider (mobile-twilio) DOES need VoicePlatform for platform-managed voice calls — it must convert between audio (Media Streams) and text (ConversationHandler). mobile-twilio depends on voice-spi and injects VoicePlatformService to obtain STT/TTS. Voicemail transcription (D7) also uses VoicePlatform STT, regardless of provider. The dependency is: mobile-twilio → voice-spi (Maven dependency), mobile-dial → no voice dependency. MobilePlatform SPI (mobile-spi) has no dependency on voice-spi.
**Trade-offs:** Provider modules have different dependency profiles. This is expected — each provider wraps a different backend with different capabilities.
**Sources:** VoicePlatform SPI (voice-spi), Dial native STT/TTS, Twilio Media Streams
**Exploration:** quick — surfaced by reviewer as implicit decision
**Status:** captured (from R1-11)

## D13: @SimulationEligible on MobilePlatform

**Choice:** MobilePlatform annotated with @SimulationEligible, capabilities array matching capability method names
**Alternatives:**
- No annotation — hand-written simulation code per SPI (pre-ADR-0011 pattern)
**Rationale:** ADR-0011 mandates @SimulationEligible for all platform SPIs. Every existing platform SPI uses it: VoicePlatform, BankPlatform, ChatPlatform (with platform#375 recursive wrapper generation for capability-based SPIs), CalendarPlatform, EmailPlatform, DocumentPlatform, ContactsPlatform, CommercePlatform, TravelPlatform, LocationPlatform, ProjectPlatform, HealthPlatform. MobilePlatform is a capability-based SPI (like ChatPlatform), so it requires the `capabilities` attribute listing capability method names. The build-time decorator generator (SimulationDecoratorProcessor) produces recursive wrapper classes for each capability using dotted qualified names (e.g. `mobile-platform.sms.send`, `mobile-platform.voiceCall.initiate`). The `capabilities` array must match the capability accessor method names on MobilePlatform exactly — this affects SPI interface design.
**Trade-offs:** Depends on the simulation framework being stable and on platform#375 (recursive wrapper generation for capability-based SPIs) being fully functional.
**Sources:** ADR-0011, SimulationEligible annotation, platform#375 recursive wrapper generation, ChatPlatform @SimulationEligible example
**Exploration:** quick — surfaced by reviewer as implicit decision
**Status:** captured (from R1-12)

## D14: Inbound message flow architecture

**Choice:** Provider modules implement WebhookInboundConnector and continue to fire InboundMessage CDI events through the existing L2 event bus
**Alternatives:**
- MobilePlatform API only, bypassing the event bus — breaks existing @ObservesAsync InboundMessage consumers who filter by type SMS/WHATSAPP
- Dual delivery (both MobilePlatform API and InboundMessage events) — creates two delivery paths for the same message, observers fire twice
- MobilePlatform internally implements WebhookInboundConnector — duplicates the L2 pattern inside the SPI
**Rationale:** The existing inbound architecture (L2) works: external platforms POST webhooks → WebhookInboundConnector verifies HMAC → InboundConnectorService fires InboundMessage via Event.fireAsync() → observers handle asynchronously. mobile-twilio's inbound SMS/WhatsApp handling follows this exact pattern — the provider module contributes WebhookInboundConnector implementations that fire InboundMessage events. The existing channelType contract is preserved: mobile-twilio's SMS inbound returns channelType "sms" and WhatsApp returns "whatsapp", maintaining compatibility with notification-bridge, CloudEvents adapter (InboundConnectorTypes.SMS, InboundConnectorTypes.WHATSAPP), and any @ObservesAsync InboundMessage consumers. MobilePlatform adds structure to the OUTBOUND side (capability-based send/call) — it does not change the inbound event model. The absorption (D5) replaces the old connector CLASSES but preserves the event flow CONTRACT.
**Trade-offs:** Inbound and outbound use different patterns: inbound is event-driven (L2 InboundMessage), outbound is capability-driven (MobilePlatform SPI). This asymmetry is intentional and follows the existing architecture — all platform SPIs use L2 for inbound events.
**Sources:** WebhookInboundConnector pattern, InboundConnectorService, InboundConnectorTypes (SMS, WHATSAPP), ConnectorsCloudEventAdapter, notification-bridge channelType contract
**Exploration:** quick — surfaced by reviewer as implicit decision
**Status:** captured (from R1-13)
