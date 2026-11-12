# MobilePlatform SPI — Design Decisions

## D1: Voice call abstraction model

**Choice:** Instruction-based — agent provides a goal/instruction, provider's AI handles the conversation autonomously, agent receives transcript and outcome
**Alternatives:**
- Audio-streaming — agent manages TTS/STT turn-by-turn via bidirectional audio channel. More powerful but impractical for MCP tools and playbooks; deferred to v2 as a separate capability
- Both from day one — instruction + streaming capabilities. Larger scope without clear v1 need
**Rationale:** Dial's API is natively instruction-based (outboundInstruction). The agent sets a goal ("book a table for 7pm, flexible within 1 hour"), the provider's AI converses freely with the human until the goal is achieved or the call ends. Nothing is scripted. This maps cleanly to MCP tools and playbook steps. Raw audio streaming is a specialised use case for v2.
**Trade-offs:** Cannot do real-time agent-controlled conversation in v1. Agents that need turn-by-turn control must wait for a VoiceCallStreaming capability.
**Sources:** Dial API docs (docs.getdial.ai), design-brief.md voice architecture section, TravelPlatform SPI pattern
**Exploration:** quick
**Status:** captured

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

## D3: mobile-ref voice call simulation

**Choice:** Dual-mode — canned scripts by default, LLM-driven by profile
**Alternatives:**
- Canned-only — simpler but no path to realistic testing
- LLM-only — richer but can't run in CI without an LLM provider
- Pluggable handler SPI — flexible but over-engineered for a ref module
**Rationale:** Tests need deterministic pre-canned conversations both ways (caller and responder). But for dev/demo, both sides should be real LLM conversations. The restaurant responder gets an artificial diary in its instruction (e.g. "all times and tables are currently available, agree to anything asked"). Profile-gated — default is canned, LLM mode behind a Quarkus profile.
**Trade-offs:** Two code paths in mobile-ref. Mitigated by the canned path being simple transcript assembly and the LLM path delegating to an injected LLM service.
**Sources:** Existing ref pattern (deterministic default, richer modes behind profiles), voice-ref WAV byte simulation
**Exploration:** quick
**Status:** captured

## D4: Twilio in v1 scope

**Choice:** Include mobile-twilio in v1 alongside mobile-dial
**Alternatives:**
- Defer Twilio to v2 — smaller scope, but delays SPI abstraction validation across providers
**Rationale:** Having two providers from day one proves the SPI abstraction works across fundamentally different backends. Dial is instruction-based (provider handles conversation), Twilio requires building the conversation loop (Media Streams WebSocket + VoicePlatform STT/TTS + LLM). This validates the SPI is a real abstraction, not a Dial wrapper.
**Trade-offs:** Twilio's voice implementation is substantially more complex — it must build the conversation layer that Dial provides natively (receive audio → STT → LLM → TTS → send audio). For SMS and messaging, Twilio is straightforward.
**Depends on:** D1 (instruction-based model — Twilio must implement this on top of raw audio)
**Sources:** Existing TwilioSmsConnector in codebase, Twilio Media Streams docs
**Exploration:** quick
**Status:** captured

## D5: Legacy connector absorption

**Choice:** Absorb TwilioSmsConnector and WhatsAppInboundConnector into mobile-twilio
**Alternatives:**
- Leave in place as parallel path — less risk but two ways to send SMS, confusing for consumers
- Deprecate without deleting — gradual migration but defers cleanup
**Rationale:** Clean break. One path for all telephony. The old connectors use the fire-and-forget Connector SPI, not the capability-based platform pattern. Moving their logic into mobile-twilio's Sms and Messaging capabilities eliminates duplication and gives consumers a single API.
**Trade-offs:** Breaking change for any consumer using TwilioSmsConnector or WhatsAppInboundConnector directly. Mitigated by pre-release stage — no external consumers yet.
**Depends on:** D4 (Twilio in v1)
**Sources:** TwilioSmsConnector.java, WhatsAppInboundConnector.java, notification-bridge module
**Exploration:** quick
**Status:** captured

## D6: Numbrrs in v1 scope

**Choice:** Include mobile-numbrrs in v1
**Alternatives:**
- Defer — two providers already validate the SPI
**Rationale:** Three providers proves the SPI scales. Numbrrs is SMS-only and simple to implement — a third data point for SPI validation with minimal effort.
**Trade-offs:** Additional module to maintain. Numbrrs is UK-only and SMS-only, so limited capability coverage.
**Sources:** Numbrrs API (numbrrs.com), issue #206
**Exploration:** quick
**Status:** captured

## D7: Voicemail transcription timing

**Choice:** Configurable — provider config flag for eager vs lazy transcription
**Alternatives:**
- Lazy only — simpler but no option for pre-transcribed voicemails
- Eager only — wastes resources on unread voicemails
**Rationale:** Different deployments have different needs. A config flag (`casehub.mobile.voicemail.transcribe-on-receive=true|false`, default false/lazy) lets operators choose. Lazy delegates to VoicePlatform STT on access; eager triggers transcription on inbound voicemail event.
**Trade-offs:** Two code paths for voicemail transcription. Mitigated by both paths using the same VoicePlatform STT — the only difference is when.
**Sources:** VoicePlatform STT integration, EmailPlatform lazy attachment content pattern
**Exploration:** quick
**Status:** captured

## D8: Two call modes — provider-managed and platform-managed

**Choice:** SPI supports both provider-managed (simple instruction) and platform-managed (agentic with mid-call tool access) voice calls
**Alternatives:**
- Platform-managed only — simpler SPI but can't use Dial's simple instruction mode for easy cases
- Provider-managed only — misses the key value of agentic calls (mid-call decisions, human-in-the-loop)
**Rationale:** Simple calls ("book a table for 7pm") can use Dial's built-in AI via `POST /calls`. Complex calls where the agent needs mid-call tool access (check with human via work queue, call another API) use Dial's self-hosted LLM WebSocket — the platform's agent handles conversation logic with full tool access. Twilio's Media Streams provides the same agentic mode. The SPI abstracts both behind a single call interface — if tools/callbacks are provided, the provider uses platform-managed mode; otherwise provider-managed.
**Trade-offs:** Platform-managed mode is substantially more complex: requires WebSocket integration, LLM session management, and tool dispatch mid-call. Provider-managed mode is simple REST.
**Depends on:** D1 (instruction-based model), D4 (Twilio in v1 — validates both modes work across providers)
**Sources:** Dial self-hosted LLM WebSocket protocol (wss://host/path/call_id, HMAC-SHA256 auth, response_required/response messages), Twilio Media Streams WebSocket
**Exploration:** quick
**Status:** captured

## D9: Agentic call conversation management

**Choice:** CallAgent SPI callback — connectors handles transport, platform provides the agent
**Alternatives:**
- CDI event-driven — looser coupling but complex event flow for real-time conversation
- Built-in conversation loop — breaks the "pure delivery infrastructure" rule
**Rationale:** Connectors stays pure delivery. `CallAgent` is an SPI: `onTranscriptUpdate(transcript) → AgentResponse` (text to speak, or tool action like "pause and check with human"). The platform wires in the LLM agent with tools. Connectors handles the WebSocket transport (Dial or Twilio), converts between provider protocol and CallAgent callbacks.
**Trade-offs:** Platform must implement CallAgent — connectors can't run agentic calls standalone. For mobile-ref testing, a simple canned CallAgent implementation handles this.
**Depends on:** D8 (platform-managed calls), D1 (instruction-based — CallAgent receives text, not audio)
**Sources:** Connector SPI pattern (connectors provides interface, platform implements), InboundConnector CDI event pattern
**Exploration:** quick
**Status:** captured
