## D1: Epic scope — all three channels plus voice stub

**Choice:** Include all three channels (WhatsApp voice note enhancement, Gmail push, voice message intake) with a full VoicePlatform SPI stub. Voice implementation migrates from agency later (#187).
**Alternatives:**
- Items 1 and 2 only — defer voice until #187 lands. Lower risk but delays the SPI surface definition.
- Gmail push only — narrowest scope. Leaves WhatsApp voice notes and pipeline for separate issues.
**Rationale:** Defining the full API surface now means #187 becomes a migration, not a design exercise. The stub ref implementation lets the shared pipeline be built and tested end-to-end without waiting for the real speech backend.
**Trade-offs:** Risk of SPI divergence if #187's design requirements differ from what we define. Mitigated by basing the SPI on agency's existing speech-api (33 types, battle-tested).
**Sources:** agency/speech-api (33 types across 9 families), connectors issue #187 (VoicePlatform SPI epic, still open)
**Exploration:** quick
**Status:** captured

## D2: Channel-agnostic STT step type for audio transcription

**Choice:** Provide a channel-agnostic STT step type (`@Plugin(streaming=true)`) that consuming applications can compose into audio processing playbooks via the platform playbook engine. All audio-producing connectors (WhatsApp, Signal, Discord) download attachments inline and deliver `Attachment` with audio content types. The STT step transcribes audio bytes regardless of source channel.
**Alternatives:**
- WhatsApp-specific first — build transcription into WhatsApp flow only, refactor to shared later. Guaranteed refactor.
- Observer-based pipeline — CDI observer detects audio attachments and enriches. Fully decoupled but adds a second event hop.
**Rationale:** All three audio-producing connectors download attachments inline (WhatsApp via D4's async download, Signal via `SignalInboundConnector.handleMessage()`, Discord via `DiscordInboundConnector.downloadAndDeliver()`). All produce `Attachment` with audio content types. The STT step is identical regardless of source channel — it takes audio bytes and calls `VoicePlatform.speechToText()`. Consuming applications compose this step into their own playbooks (D12), with per-application retry/timeout requirements.
**Trade-offs:** Connectors provides only the step atom — consuming applications must author playbook YAML. This is the correct trade-off per D12: audio processing pipelines differ by application.
**Sources:** SignalInboundConnector.java (downloads attachments inline), DiscordInboundConnector.java (same pattern), WhatsAppInboundConnector.java (D4: async download with retry), D12 (playbook YAML ownership)
**Exploration:** quick
**Status:** revised (R2-03: updated language to match post-revision scope — connectors provides a step type, not a pipeline)

## D3: Pipeline architecture — playbook orchestration with native streaming steps

**Choice:** Use the platform playbook engine for orchestration (routing, retry, timeout, error handling, composition) with a custom native step type for the data plane (STT transcription).
**Alternatives:**
- Simple CDI content-type router in connectors — no playbook dependency, less flexible but zero orchestration overhead.
- Defer pipeline architecture — focus on raw capabilities, design pipeline separately.
**Rationale:** Playbooks already have CloudEvent triggers, match/cases for content-type routing, retry with backoff, timeout, channels with backpressure. The step compiler is extensible — custom step types handle streaming efficiently as plain Java. Clean separation: playbook handles orchestration plane, native steps handle data plane.
**Trade-offs:** Playbook dependency for pipeline consumers. Mitigated because playbooks are already a platform primitive. The STT step type (stt-transcribe) needs to be built as a step compiler extension.
**Sources:** platform/yaml-step-runtime (DecoratorChain, StructuralStepEvaluator), platform/playbook-compiler (extensible register()), playbooks/concurrency-patterns/combined-pipeline.playbook.yaml
**Exploration:** deep-analysis
**Status:** captured

## D4: WhatsApp media download — async on virtual thread with explicit failure handling

**Choice:** Download WhatsApp voice note media asynchronously on a virtual thread, matching Discord's pattern. Return 200 to Meta immediately, download + process on a virtual thread, deliver to sink when ready. On download failure: retry with exponential backoff (3 attempts, 1s/2s/4s delays). On final failure: fire InboundMessage with metadata `download-failed=true` and `media-url=<url>` — consumer decides whether to retry or alert.
**Alternatives:**
- Sync in handler — download before returning WebhookResult. Simpler but risks Meta webhook timeouts (20s limit) on slow connections.
- Deferred via CDI event — deliver InboundMessage with media URL metadata, let pipeline download later. Most decoupled but InboundMessage arrives incomplete.
**Rationale:** Meta retries on slow webhook responses. Discord already uses this pattern successfully. Virtual threads make async I/O natural. The async download must happen immediately after returning 200 — WhatsApp media URLs expire after a configurable period (default 5 minutes). Retry with exponential backoff handles transient network failures. Final failure fires an InboundMessage with failure metadata rather than silently dropping the voice note, enabling the consumer (e.g. casehub-life) to take corrective action (alert user, queue for manual review).
**Trade-offs:** The webhook returns 200 before processing completes — if download fails after all retries, the voice note content is lost (only metadata delivered). Mitigated by immediate download (within seconds of webhook receipt, well within URL expiry window), 3-attempt retry, and explicit failure metadata in the delivered InboundMessage.
**Sources:** DiscordInboundConnector.java lines 144-158 (virtual thread attachment download with failure counting), WhatsApp Cloud API docs (media URL expiration)
**Exploration:** quick
**Status:** revised (R1-08: added explicit retry strategy, failure delivery, URL expiration awareness)

## D5: Gmail push — push-only via WebhookInboundConnector

**Choice:** Support Gmail push notifications via Google Cloud Pub/Sub push subscription, implemented as a `WebhookInboundConnector`. No pull subscription mode — IMAP IDLE (existing `email-inbound` module) already serves that role and is the simpler default.
**Alternatives:**
- Push + pull modes — doubles the Gmail inbound surface for marginal benefit. Pull via Pub/Sub is functionally equivalent to IMAP IDLE (both require a running process with a persistent connection). Eliminated.
- IMAP IDLE only — no Gmail push. Simpler, already works. Valid for single-account deployments. Does not serve webhook-preferred or multi-account deployments.
- Push only (chosen) — WebhookInboundConnector receives Pub/Sub push notifications. Stateless, event-driven, supports label-specific watches.
**Rationale:** Gmail push provides a webhook-based alternative to IMAP IDLE that works in stateless/multi-account deployments without persistent connections. IMAP IDLE remains the simpler default and requires no GCP infrastructure. Both share `GoogleEmailPlatform` for message fetching — only the notification mechanism differs. Pull subscription mode dropped because it provides no advantage over IMAP IDLE: both require a running process, both maintain a connection. Push-only halves the implementation surface.
**Trade-offs:** Gmail push requires a GCP project (Pub/Sub topic, subscription, IAM) and a publicly reachable HTTPS endpoint. IMAP IDLE requires only outbound TCP. The choice is deployment-dependent. Both produce `InboundMessage` events through the standard CDI async event bus.
**Sources:** WebhookInboundConnector.java (push pattern), EmailInboundConnector.java (IMAP IDLE — existing, working), GoogleEmailPlatform in email-google (reused for message fetch), ARC42STORIES §9.3 C3 (IMAP IDLE sub-second latency)
**Exploration:** quick
**Status:** revised (R1-05: dropped pull subscription mode, acknowledged IMAP IDLE as default)

## D6: VoicePlatform SPI — STT + TTS capability surface

**Choice:** Define two capability families in the VoicePlatform SPI: `SpeechToText` and `TextToSpeech`. Avatar/cognition stays in agency. Denoising, VAD, SpeakerID, Diarization, TextCleanup, and LipSync deferred to #187 — to be added only when a second consumer exists outside agency.
**Alternatives:**
- Full 8-family surface (original choice) — defines all 8 families from agency's speech-api. Gives #187 a complete migration target but adds 6 capability families with no consumer outside agency. LipSync and PhonemeAligner are avatar-specific (consumed only by `speech-ws` in agency), violating connectors' "no domain knowledge" library rule. Denoising, VAD, Diarization, SpeakerID are audio processing capabilities, not channel/delivery capabilities — they don't interface with external systems.
- STT + TTS + SpeakerID — core capabilities plus speaker identification for actor recognition. Defers 5 families.
- STT + TTS only (chosen) — minimal surface matching issue #187's explicit scope ("SpeechToText — audio stream/bytes → transcribed text" and "TextToSpeech — text → audio stream/bytes"). Follows the platform SPI pattern: VoicePlatform interfaces with speech services (Whisper, Google Cloud Speech), analogous to ChatPlatform interfacing with chat systems.
**Rationale:** Every existing platform SPI in connectors is an interface to an external system's channel surface (ChatPlatform → chat systems, EmailPlatform → email systems, BankPlatform → banking systems). VoicePlatform with STT + TTS fits this pattern — it interfaces with speech-to-text and text-to-speech services. Denoising, VAD, diarization, speaker identification, text cleanup, and lip sync are processing capabilities, not channel interfaces. Adding them prematurely creates domain coupling (especially LipSync/PhonemeAligner which are avatar-specific) and violates the connectors boundary rule. Capabilities can be added incrementally to the SPI as sub-interfaces when a concrete consumer outside agency requires them.
**Trade-offs:** Issue #187 may need to add more capability sub-interfaces when the full migration happens. Mitigated by the capability sub-interface pattern — new families add new interfaces without breaking existing ones.
**Sources:** issue #187 (explicitly scopes to STT + TTS), existing platform SPI patterns (ChatPlatform — 10 capabilities, BankPlatform — 2 capabilities), agency speech-api (33 types across 9 families — 8 excluding Avatar/Cognition)
**Exploration:** quick
**Status:** revised (R1-02: scoped to STT + TTS matching issue #187, deferred 6 families)

## D7: Use platform's streaming step API with automatic pipe fusion

**Choice:** Implement stt-transcribe as a `@Plugin(streaming=true)` step type using the platform's shipping `StreamingStep<I, O>` API. Media download is handled inline by each connector (WhatsApp via D4, Signal and Discord via existing code). The `FusionPass` automatically detects and fuses consecutive type-compatible streaming steps into a single `FusedPipelineStep` — applicable when consuming applications compose multiple streaming steps (e.g. denoise → STT in future pipelines). For #188's single-step STT pipeline, the playbook engine still provides per-step retry/timeout decorators.
**Alternatives:**
- Custom pipeline framework in connectors — redundant given the platform already ships this.
- Plain CDI service chain — no playbook integration, no automatic retry/timeout/error handling.
**Rationale:** The platform's streaming step infrastructure (platform#680, now landed) provides exactly what this pipeline needs: typed transforms (`StreamingStep<I, O>`), automatic fusion by the compiler (`FusionPass`), and per-stage `timeout`/`retry`/`delay` decorators. The "streaming" in `StreamingStep` refers to step-to-step data flow (typed piping between stages), not real-time content streaming — the input to a step can be `byte[]` (fully materialized) and the typed composition still provides value via fusion and per-stage error handling. The runtime evaluator (`StructuralStepEvaluator`) provides `AutoCloseable` cleanup for intermediate data values on failure — if a step produces a closeable resource (e.g. `InputStream`) and the next step in the chain fails, the resource is closed. This cleanup applies to data flowing through the pipeline, not to step instances. Using the platform's infrastructure means zero custom pipeline code in connectors.
**Trade-offs:** Dependency on `yaml-plugin-api` for the `@Plugin` annotation and `StreamingStep` interface. Acceptable — this is the platform's standard step authoring API, analogous to the `quarkus-mcp-server-core` dependency for MCP tools.
**Depends on:** D3 (playbook orchestration architecture)
**Sources:** platform StreamingStep.java, FusionPass.java, StructuralStepEvaluator.java (lines 194-205, AutoCloseable cleanup in fused chain evaluation), StreamingPipelineIntegrationTest.java, @Plugin(streaming=true) annotation
**Exploration:** deep-analysis
**Status:** revised (R1-11: clarified streaming/AutoCloseable semantics; R2-02: removed stale media-download reference, clarified single-step pipeline for #188)

## D8: VoicePlatform SPI structure — connectors capability pattern

**Choice:** Single `VoicePlatform` interface with capability sub-interfaces matching the connectors platform SPI pattern: `speechToText()`, `textToSpeech()`. `supports(Class)` introspection. `@SimulationEligible`.
**Alternatives:**
- Flat interfaces, no platform wrapper — mirror agency's speech-api structure. Simpler but inconsistent with other platform SPIs.
- Hybrid — standalone interfaces AND a VoicePlatform aggregator. More flexible but more types.
**Rationale:** Consistency with ChatPlatform (10 capabilities), BankPlatform (2 capabilities — AccountInformation + PaymentInitiation), DocumentPlatform, etc. The capability sub-interface pattern with `supports()` introspection enables the simulation framework and graceful degradation. Starting with 2 capabilities (matching BankPlatform's pattern) and adding more as needed is consistent with the platform's incremental growth model.
**Trade-offs:** Agency's speech-api types need to be adapted to fit the capability pattern. With only STT + TTS, the adaptation is straightforward.
**Sources:** ChatPlatform (10 capabilities — Messaging, Threading, Discovery, Reactions, Presence, Members, ChannelManagement, MemberManagement, MessageHistory, Commands), BankPlatform (2 capabilities — AccountInformation + PaymentInitiation), agency speech-api (33 types, 9 families)
**Exploration:** quick
**Status:** revised (R1-10: fixed ChatPlatform count from 9 to 10; R1-02: simplified to STT + TTS matching D6 revision)

## D9: Module structure — 4 new modules

**Choice:** Create 4 new modules in connectors:

- `voice-spi` — VoicePlatform SPI with `SpeechToText` + `TextToSpeech` capability sub-interfaces + model types
- `voice-ref` — in-memory reference VoicePlatform with stubs for STT + TTS (canned transcriptions, placeholder audio)
- `email-google-inbound` — Gmail push via Pub/Sub (WebhookInboundConnector only), depends on email-google for message fetch
- `voice-pipeline-steps` — STT `@Plugin(streaming=true)` step using VoicePlatform SPI

Existing module modifications:
- `webhook` — enhance WhatsAppInboundConnector with voice note media download (async virtual thread, retry, failure metadata)

Not included (deferred or eliminated):
- `whatsapp-inbound-media` — eliminated; media download integrated into existing WhatsAppInboundConnector in the webhook module
- `signal-inbound-media` — eliminated; Signal already downloads attachments inline in `SignalInboundConnector.handleMessage()`
- 7 migration modules (voice-sherpa + 5 native JARs + absorbed speech-api types) — deferred to #187; SPI in connectors, implementation stays in agency until migration

**Alternatives:**
- 13 modules (original) — 6 new + 7 migrated from agency. Unnecessary scope for #188 given D6 scoped to STT + TTS and D10 deferred migration.
- 6 modules only, no migration — what the original D9 proposed as an alternative. Close to the revised choice.
**Rationale:** Each module has one clear purpose. Eliminated modules were either redundant with existing connectors code (Signal already downloads attachments, WhatsApp media download belongs in the webhook module) or premature (migration modules depend on #187). 4 new modules in a 40+ module project is a modest addition.
**Trade-offs:** Less upfront migration than the original proposal. Mitigated by the SPI being designed for the migration (#187 becomes a pure migration exercise against an already-defined contract).
**Sources:** SignalInboundConnector.java (already handles attachments inline), DiscordInboundConnector.java (download pattern in existing module, not separate), existing module structure in parent POM
**Exploration:** quick
**Status:** revised (R1-06: reduced from 13 to 4 modules; R1-02: scoped to STT + TTS; R1-07: deferred migration)

## D10: VoicePlatform SPI in connectors — implementation migration deferred to #187

**Choice:** Define the VoicePlatform SPI (`voice-spi`) and reference implementation (`voice-ref`) in connectors. The production implementation (`speech-sherpa` with native library modules) stays in agency and implements the connectors SPI via a dependency inversion: agency depends on connectors' `voice-spi`. Full migration of `speech-sherpa` and its native libraries into connectors happens in #187.
**Alternatives:**
- Full migration now (original choice) — migrate speech-sherpa + 5 native JARs + speech-api types into connectors. Moves native build infrastructure (C++, CMake, OS/architecture matrix) into connectors CI, which is not configured for it. The migration depends on D6's scope (now STT + TTS only, reducing the surface) and D8's structure (now 2 capabilities, simplifying the SPI). Cross-repo change across connectors and agency simultaneously.
- SPI in connectors, impl stays in agency permanently — breaks the pattern of every other platform SPI having its implementation in connectors. Acceptable as a staging strategy, not as a final state.
**Rationale:** The right architectural end-state IS implementation in connectors (matching ChatPlatform, EmailPlatform, BankPlatform patterns). But achieving it requires: (1) native build toolchain in connectors CI, (2) cross-platform test infrastructure for 5 OS/architecture targets, (3) coordinated cross-repo changes in both connectors and agency. These are real infrastructure concerns that #187 should address with dedicated focus, not #188 which is about three inbound channels. The dependency inversion (agency implements connectors SPI) works cleanly: `speech-sherpa` implements `VoicePlatform`, consuming applications depend on `voice-spi`, and the native libraries stay where their CI already works.
**Trade-offs:** Temporarily inverted dependency direction — agency implements a connectors SPI while the implementation module lives in agency. This is unusual but not unprecedented (any application can implement a platform SPI). Mitigated by #187 being the explicit migration ticket.
**Depends on:** D8 (VoicePlatform SPI structure), D6 (SPI scope — now STT + TTS only)
**Sources:** agency/speech-sherpa (implementation), agency/speech-api (interfaces), agency/speech-sherpa-native-* (5 platform-specific native library JARs), existing platform SPI pattern (SPI in connectors, impl in connectors)
**Exploration:** quick
**Status:** revised (R1-07: deferred full migration to #187, SPI-only in #188)

## D11: Voice is a delivery channel, not a processing toolkit

**Choice:** VoicePlatform is an interface to speech services (STT, TTS providers like Whisper, Google Cloud Speech), following the same pattern as ChatPlatform (interface to chat systems) and EmailPlatform (interface to email systems). Audio processing capabilities (denoising, VAD, diarization, speaker identification, text cleanup, lip sync) are NOT part of the VoicePlatform SPI.
**Alternatives:**
- Voice as processing toolkit — VoicePlatform includes all audio processing capabilities. Turns connectors into an audio processing library with domain knowledge about avatar rendering (LipSync), speaker analysis (SpeakerID, Diarization), and signal processing (Denoising, VAD). Violates connectors' library rule ("no business logic, no domain knowledge").
- Voice as hybrid — VoicePlatform has STT + TTS as channel capabilities, plus a separate AudioProcessing SPI for denoising/VAD. Splits the concern but still places processing capabilities in a delivery library.
**Rationale:** Every existing platform SPI in connectors interfaces with an external system's surface: ChatPlatform → chat systems, EmailPlatform → email systems, BankPlatform → banking systems, CalendarPlatform → calendar services. Processing capabilities (denoising audio, identifying speakers, aligning phonemes) don't interface with external systems — they're local transforms. Placing them in connectors would be like adding an image processing library because images arrive via WhatsApp.
**Trade-offs:** Audio processing capabilities needed by consuming applications must be sourced from elsewhere (e.g., agency retains them, or a dedicated audio-processing library is created). For #188's voice message intake, STT via VoicePlatform is sufficient — the pipeline receives downloaded audio bytes and transcribes them.
**Sources:** Connectors ARC42STORIES §1 ("Platform library: outbound and inbound message delivery"), §2 ("No business logic, routing, or orchestration"), existing platform SPI patterns
**Exploration:** quick (surfaced by reviewer R1-09, made explicit)
**Status:** captured

## D12: Playbook YAML ownership — consuming applications, not connectors

**Choice:** Connectors provides the `@Plugin(streaming=true)` stt-transcribe step type as a CDI bean. Media download is handled inline by each connector, not as a pipeline step. The playbook YAML that composes step types into processing pipelines lives in consuming applications (casehub-life, casehub-clinical, etc.), not in connectors.
**Alternatives:**
- Pipeline YAML in connectors — connectors ships a shared pipeline playbook. Violates the library rule ("no orchestration"). Connectors provides building blocks; orchestration is a caller concern.
- Pipeline YAML in a separate shared module — new module category. Creates a shared pipeline that all consumers inherit. Reduces flexibility; each application's audio processing needs may differ.
- Step types + YAML in consuming apps (chosen) — connectors provides the atoms (step types), apps compose the molecules (playbooks). Standard separation in the platform.
**Rationale:** Step types are analogous to MCP tools: CDI beans with a platform annotation (`@Plugin` vs `@Tool`) that declares them as integration points. The MCP module already hosts `@Tool` beans with a `quarkus-mcp-server-core` dependency. Step types follow the same pattern with a `yaml-plugin-api` dependency. Neither @Tool beans nor @Plugin beans contain orchestration — they're leaf functions invoked by their respective runtimes.
**Trade-offs:** Each consuming application must author its own pipeline YAML. This is the correct trade-off — audio processing pipelines differ by application (life needs transcription for voice notes; clinical may need diarization for multi-speaker consultations; each has different retry/timeout requirements).
**Sources:** MCP module pattern (mcp/src with @Tool @Blocking beans depending on quarkus-mcp-server-core), platform playbook architecture (steps in plugin modules, YAML in consuming apps)
**Exploration:** quick (surfaced by reviewer R1-04, made explicit)
**Status:** captured
