## D1: Epic scope — all three channels plus voice stub

**Choice:** Include all three channels (WhatsApp voice note enhancement, Gmail push, voice message intake) with a full VoicePlatform SPI stub. Voice implementation migrates from agency later (#187).
**Alternatives:**
- Items 1 and 2 only — defer voice until #187 lands. Lower risk but delays the SPI surface definition.
- Gmail push only — narrowest scope. Leaves WhatsApp voice notes and pipeline for separate issues.
**Rationale:** Defining the full API surface now means #187 becomes a migration, not a design exercise. The stub ref implementation lets the shared pipeline be built and tested end-to-end without waiting for the real speech backend.
**Trade-offs:** Risk of SPI divergence if #187's design requirements differ from what we define. Mitigated by basing the SPI on agency's existing speech-api (34 types, battle-tested).
**Sources:** agency/speech-api (34 types across 9 families), connectors issue #187 (VoicePlatform SPI epic, still open)
**Exploration:** quick
**Status:** captured

## D2: Shared audio processing pipeline

**Choice:** Build a channel-agnostic audio processing pipeline that any inbound connector can feed into. WhatsApp and Signal both route audio attachments through the same STT path.
**Alternatives:**
- WhatsApp-specific first — build transcription into WhatsApp flow only, refactor to shared later. Guaranteed refactor.
- Observer-based pipeline — CDI observer detects audio attachments and enriches. Fully decoupled but adds a second event hop.
**Rationale:** Signal already downloads attachments. WhatsApp needs to. The future voice connector will too. All produce `Attachment` with audio content types. The STT stage is identical regardless of source channel.
**Trade-offs:** Slightly more upfront design work. Signal inbound needs modification to feed into the pipeline (currently delivers raw attachments without processing).
**Sources:** SignalInboundConnector.java (downloads attachments inline), DiscordInboundConnector.java (same pattern), WhatsAppInboundConnector.java (currently URL-only for media)
**Exploration:** quick
**Status:** captured

## D3: Pipeline architecture — playbook orchestration with native streaming steps

**Choice:** Use the platform playbook engine for orchestration (routing, retry, timeout, error handling, composition) with custom native step types for the data plane (binary media download, streaming STT).
**Alternatives:**
- Simple CDI content-type router in connectors — no playbook dependency, less flexible but zero orchestration overhead.
- Defer pipeline architecture — focus on raw capabilities, design pipeline separately.
**Rationale:** Playbooks already have CloudEvent triggers, match/cases for content-type routing, retry with backoff, timeout, channels with backpressure. The step compiler is extensible — custom step types handle streaming efficiently as plain Java. Clean separation: playbook handles orchestration plane, native steps handle data plane.
**Trade-offs:** Playbook dependency for pipeline consumers. Mitigated because playbooks are already a platform primitive. Streaming step types need to be built (media-download, stt-transcribe) as step compiler extensions.
**Sources:** platform/yaml-step-runtime (DecoratorChain, StructuralStepEvaluator), platform/playbook-compiler (extensible register()), playbooks/concurrency-patterns/combined-pipeline.playbook.yaml
**Exploration:** deep-analysis
**Status:** captured

## D4: WhatsApp media download — async on virtual thread

**Choice:** Download WhatsApp voice note media asynchronously on a virtual thread, matching Discord's pattern. Return 200 to Meta immediately, download + process on a virtual thread, deliver to sink when ready.
**Alternatives:**
- Sync in handler — download before returning WebhookResult. Simpler but risks Meta webhook timeouts (20s limit).
- Deferred via CDI event — deliver InboundMessage with media URL metadata, let pipeline download later. Most decoupled but InboundMessage arrives incomplete.
**Rationale:** Meta retries on slow webhook responses. Discord already uses this pattern successfully. Virtual threads make async I/O natural.
**Trade-offs:** The webhook returns 200 before processing completes — if download fails, there's no HTTP-level retry from Meta. Acceptable because Meta's retry is unreliable anyway and the download can retry internally.
**Sources:** DiscordInboundConnector.java lines 131-139 (virtual thread attachment download pattern)
**Exploration:** quick
**Status:** captured

## D5: Gmail push — both push and pull modes, configurable

**Choice:** Support both Pub/Sub push subscription (WebhookInboundConnector) and pull subscription (InboundConnector) modes, configurable per deployment.
**Alternatives:**
- Push only — matches WebhookInboundConnector pattern, requires public HTTPS endpoint.
- Pull only — matches InboundConnector pattern like IMAP IDLE, no public endpoint needed.
**Rationale:** Different deployments have different network constraints. Push is lower latency but requires a publicly reachable endpoint. Pull works behind firewalls. Both reuse GoogleEmailPlatform for message fetching.
**Trade-offs:** Doubles the Gmail inbound implementation surface. Mitigated by sharing the message fetching logic via GoogleEmailPlatform — only the notification receipt mechanism differs.
**Sources:** WebhookInboundConnector.java (push pattern), InboundConnector.java (pull pattern), GoogleEmailPlatform in email-google (reused for message fetch)
**Exploration:** quick
**Status:** captured

## D6: VoicePlatform SPI — full 8-family capability surface

**Choice:** Define all 8 capability families from agency's speech-api in the VoicePlatform SPI: STT, TTS, Denoising, VAD, SpeakerID, Diarization, TextCleanup, LipSync. Avatar/cognition stays in agency. Ref implementation stubs all capabilities.
**Alternatives:**
- STT + TTS + SpeakerID only — core capabilities plus speaker identification for actor recognition. Defers 5 families.
- STT + TTS only — minimal surface for the audio pipeline. Everything else added during #187.
**Rationale:** Defining the full surface now gives #187 a complete migration target. Avoids incremental API churn. The SPI design is based on agency's battle-tested speech-api, reducing divergence risk.
**Trade-offs:** Larger upfront API design. If agency's implementation evolves before migration, the SPI may need updates. Mitigated by keeping the SPI as capability sub-interfaces — individual families can be revised independently.
**Sources:** agency/speech-api (SpeechToTextService, TextToSpeechService, SpeechDenoiser, VoiceActivityFilter, SpeakerEmbeddingExtractor, SpeakerDiarizationService, TextFilter/CleanupConfig, PhonemeAligner), existing platform SPI patterns (ChatPlatform, BankPlatform with capability sub-interfaces and supports())
**Exploration:** quick
**Status:** captured

## D7: Use platform's streaming step API with automatic pipe fusion

**Choice:** Implement media-download, stt-transcribe, denoise, and VAD as `@Plugin(streaming=true)` step types using the platform's shipping `StreamingStep<I, O>` API. The `FusionPass` automatically detects and fuses consecutive type-compatible steps into a single `FusedPipelineStep` — near-native piped execution with `AutoCloseable` resource cleanup.
**Alternatives:**
- Custom pipeline framework in connectors — redundant given the platform already ships this.
- Plain CDI service chain — no playbook integration, no automatic retry/timeout/error handling.
**Rationale:** The platform's streaming step infrastructure (platform#680, now landed) provides exactly what this pipeline needs: typed transforms (`StreamingStep<I, O>`), automatic fusion by the compiler (`FusionPass`), per-stage `timeout`/`retry`/`delay` decorators, and `AutoCloseable` resource cleanup on failure. Using it means zero custom pipeline code in connectors.
**Trade-offs:** Dependency on `yaml-plugin-api` for the `@Plugin` annotation and `StreamingStep` interface. Acceptable — this is the platform's standard step authoring API.
**Depends on:** D3 (playbook orchestration architecture)
**Sources:** platform StreamingStep.java, FusionPass.java, StreamingPipelineIntegrationTest.java, @Plugin(streaming=true) annotation
**Exploration:** deep-analysis
**Status:** captured

## D8: VoicePlatform SPI structure — connectors capability pattern

**Choice:** Single `VoicePlatform` interface with capability sub-interfaces matching the connectors platform SPI pattern: `speechToText()`, `textToSpeech()`, `denoising()`, `voiceActivityDetection()`, `speakerIdentification()`, `diarization()`, `textCleanup()`, `lipSync()`. `supports(Class)` introspection. `@SimulationEligible`.
**Alternatives:**
- Flat interfaces, no platform wrapper — mirror agency's speech-api structure. Simpler but inconsistent with other platform SPIs.
- Hybrid — standalone interfaces AND a VoicePlatform aggregator. More flexible but more types.
**Rationale:** Consistency with ChatPlatform, BankPlatform, DocumentPlatform, etc. The capability sub-interface pattern with `supports()` introspection enables the simulation framework and graceful degradation.
**Trade-offs:** Agency's speech-api types need to be adapted to fit the capability pattern (e.g., `SpeechToTextService` becomes a capability sub-interface of `VoicePlatform`). Some types may need reshaping.
**Sources:** ChatPlatform (10 capabilities), BankPlatform (AccountInformation + PaymentInitiation), agency speech-api (34 types, 8 families)
**Exploration:** quick
**Status:** captured

## D9: Module structure — 6 new modules

**Choice:** Create 6 new Maven modules: `voice-spi` (VoicePlatform SPI + 8 capability interfaces), `voice-ref` (in-memory ref impl with stubs), `email-google-inbound` (Gmail push/pull connector), `whatsapp-inbound-media` (WhatsApp media download streaming step), `voice-pipeline-steps` (STT/denoise/VAD playbook streaming steps), `signal-inbound-media` (Signal audio pipeline step reusing voice-pipeline-steps).
**Alternatives:**
- 4 modules, consolidated — merge WhatsApp/Signal media + voice steps into one module. Less granular dependency control.
- 3 modules, minimal — media download and voice steps stay in existing modules. Muddies module boundaries.
**Rationale:** Each module has one clear purpose and can be depended on independently. `voice-pipeline-steps` is shared between WhatsApp and Signal without either depending on the other.
**Trade-offs:** More modules means more POMs and more wiring. Mitigated by the existing project structure (40+ modules already) and clear dependency graph.
**Sources:** Existing module structure in parent POM, connectors module table in CLAUDE.md
**Exploration:** quick
**Status:** captured
