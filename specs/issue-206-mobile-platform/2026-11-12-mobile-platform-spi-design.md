# MobilePlatform SPI — Design Spec

Issue: connectors#206

## Overview

MobilePlatform is a capability-based SPI for phone-number-based communication: SMS, voice calls, voicemail, and rich messaging (WhatsApp/iMessage). It unifies all telephony under one platform with six capability sub-interfaces, following the ChatPlatform composition pattern (Builder + graceful degradation).

Two voice call modes: provider-managed (fire-and-forget instruction) and platform-managed (interactive text-level conversation with mid-call tool access via ConversationHandler callback).

## Modules

| Module | Purpose | Dependencies |
|--------|---------|-------------|
| `mobile-spi` | MobilePlatform interface, capabilities, model types, NoOp default, MobilePlatformService registry | `connectors-api`, `simulation-api`, `quarkus-arc` |
| `mobile-ref` | In-memory reference: switchboard, SMS log, voicemail store. Seed data from YAML. | `mobile-spi`, `quarkus-arc` |
| `mobile-dial` | Dial (getdial.ai) provider — SMS, voice, WhatsApp/iMessage, number queries | `mobile-spi`, `connectors-api`, `platform-api` |
| `mobile-twilio` | Twilio provider — SMS, voice (Media Streams), WhatsApp. Absorbs legacy connectors. | `mobile-spi`, `connectors-api`, `platform-api`, `voice-spi` |

## Capability Sub-Interfaces

Six capabilities as inner interfaces of `MobilePlatform`. This is a new hybrid composition pattern (D11) combining:
- **Inner interfaces** (TravelPlatform style) — all capability definitions co-located within `MobilePlatform`, clean namespace (`MobilePlatform.Sms`, `MobilePlatform.VoiceCall`)
- **Builder + graceful degradation** (ChatPlatform style) — heterogeneous provider landscape (Dial: all six; Twilio: SMS + voice + messaging; future SMS-only providers) requires composed capabilities with NoOp degradation defaults

Each provider declares which capabilities it supports via `supports(Class<?>)`.

### Sms

```java
interface Sms {
    SendResult send(String to, String body);
    SendResult send(String to, String body, SmsOptions options);
    Page<SmsMessage> listMessages(PageRequest page);
}
```

`SmsOptions`: fromNumberId, mediaUrls. `SmsMessage`: id, from, to, body, status, direction (INBOUND/OUTBOUND), timestamp.

### VoiceCall

Two modes at the SPI surface — both text-level, no audio exposed.

```java
interface VoiceCall {
    // Provider-managed: provider's AI handles the conversation
    Call initiate(String to, String instruction);
    Call initiate(String to, String instruction, CallOptions options);

    // Platform-managed: caller provides conversation logic
    Call initiate(String to, ConversationHandler handler);
    Call initiate(String to, ConversationHandler handler, CallOptions options);

    Call getCall(String callId);
    Page<Call> listCalls(PageRequest page);
}
```

`CallOptions`: fromNumberId, language (BCP-47), maxDurationSeconds, transferTo.

`Call`: id, to, from, status (QUEUED/RINGING/IN_PROGRESS/COMPLETED/FAILED/BUSY/NO_ANSWER/CANCELLED), transcript (flat text), transcriptTurns (List\<TranscriptTurn\>), duration, startedAt, endedAt.

`TranscriptTurn`: speaker (AGENT/HUMAN/TRANSFER_TARGET), text, startMs, endMs.

`ConversationHandler`: callback interface for platform-managed calls.

```java
interface ConversationHandler {
    // Called when human finishes speaking (text from STT)
    ConversationResponse onHumanSpeech(String text, List<TranscriptTurn> fullTranscript);

    // Called when call connects
    default void onCallConnected() {}

    // Called when call ends
    default void onCallEnded(Call completedCall) {}
}
```

`ConversationResponse`: text (what to say next), with optional action.

Actions:
- **No action (null)** — normal flow: provider speaks the text, waits for human speech, then calls `onHumanSpeech` again.
- **HOLD** — provider speaks the text, then immediately re-invokes `onHumanSpeech` with the current transcript (without waiting for human speech). Use for interim status updates: the handler returns `ConversationResponse("Please hold while I check availability", HOLD)`, the provider speaks it, the handler gets another turn to perform work, and returns the final answer with no action.
- **TRANSFER(transferTo)** — provider speaks the text (if any), then transfers the call to the specified number.
- **HANG_UP** — provider speaks the text (if any), then ends the call.

The handler call is synchronous — the provider blocks while the handler performs work (check work queues, call APIs, consult a human). The handler runs on a virtual thread to avoid blocking platform threads.

### VoiceCallControl

```java
interface VoiceCallControl {
    void hangUp(String callId);
    void transfer(String callId, String transferTo);
}
```

External out-of-band call control — used by supervisors, timeout monitors, admin panels, or other system components outside the active conversation handler. For provider-managed calls (no ConversationHandler), this is the only call control mechanism.

**Relationship to ConversationResponse actions:** ConversationResponse actions (HOLD, TRANSFER, HANG_UP) are handler-initiated, within the synchronous `onHumanSpeech` turn. VoiceCallControl is externally-initiated, from any thread at any time. If VoiceCallControl is invoked while a ConversationHandler callback is in progress, VoiceCallControl wins — the provider terminates/transfers the call and the handler's `onCallEnded()` is invoked with the externally-initiated status.

### Voice Call Failure Handling

**Handler throws during `onHumanSpeech()`:** The provider catches the exception, logs it, speaks a configurable error message (default: "I'm sorry, I'm having technical difficulties. Goodbye."), and hangs up. The handler's `onCallEnded()` is invoked with FAILED status and the exception available via `Call.failureReason()`.

**WebSocket drops mid-conversation:** The provider detects the disconnection and invokes `onCallEnded()` with FAILED status. If the handler is blocked in `onHumanSpeech()`, the provider interrupts the handler's virtual thread. The handler should treat `InterruptedException` as a call termination signal.

**VoicePlatform STT/TTS unavailable for platform-managed calls (mobile-twilio):** Hard failure, not graceful degradation. `initiate()` with a ConversationHandler must fail fast with `UnsupportedOperationException` if the injected VoicePlatformService has no real STT/TTS provider (i.e. only NoOpVoicePlatform). A NoOp STT returning empty text would produce a silent call — worse than failing. The check occurs at `initiate()` time, before the call is placed.

**Handler blocks indefinitely:** No built-in timeout at the SPI level. The `maxDurationSeconds` in CallOptions applies to the overall call duration. Provider implementations enforce this via their platform's call timeout. When the call times out, the provider invokes `onCallEnded()` with COMPLETED or FAILED status.

### Voicemail

```java
interface Voicemail {
    Page<VoicemailMessage> list(PageRequest page);
    VoicemailMessage get(String voicemailId);
    void delete(String voicemailId);
    TranscriptionResult transcribe(String voicemailId);
}
```

`VoicemailMessage`: id, from, to, durationSeconds, timestamp, audioFormat, transcription (nullable — populated if eager transcription enabled).

Transcription delegates to VoicePlatform STT. Configurable eager/lazy via `casehub.mobile.voicemail.transcribe-on-receive` (default: false/lazy). Runtime dependency: requires a deployed VoicePlatform with real SpeechToText (voice-sherpa/Whisper). Without one, transcription returns empty results (graceful degradation).

### Messaging

Rich messaging for WhatsApp, iMessage, RCS.

```java
interface Messaging {
    SendResult send(String to, String body);
    SendResult send(String to, String body, MessagingOptions options);
    Page<RichMessage> listMessages(PageRequest page);
}
```

`MessagingOptions`: channel (WHATSAPP/IMESSAGE/RCS — auto-detected if omitted), fromNumberId, mediaUrls, templateId.

`RichMessage`: id, from, to, body, channel, media (List\<MediaAttachment\>), status, direction, timestamp.

### NumberInfo

Read-only number queries. No provisioning in v1.

```java
interface NumberInfo {
    List<PhoneNumber> listNumbers();
    NumberCapabilities lookup(String phoneNumber);
}
```

`PhoneNumber`: id, number (E.164), capabilities (Set\<NumberCapability\>), status (PROVISIONING/READY/FAILED).

`NumberCapability`: SMS, VOICE, WHATSAPP, IMESSAGE, RCS.

`NumberCapabilities`: number, carrier, capabilities, countryCode, numberType (MOBILE/LANDLINE/VOIP).

## MobilePlatform Interface

```java
@SimulationEligible(name = "mobile-platform",
    capabilities = {"sms", "voiceCall", "voiceCallControl", "voicemail", "messaging", "numberInfo"})
public interface MobilePlatform {

    String id();
    boolean supports(Class<?> capability);

    Sms sms();
    VoiceCall voiceCall();
    VoiceCallControl voiceCallControl();
    Voicemail voicemail();
    Messaging messaging();
    NumberInfo numberInfo();

    // Builder for composed capabilities with graceful degradation
    static Builder builder(String id) { ... }

    interface Builder {
        Builder sms(Sms sms);
        Builder voiceCall(VoiceCall voiceCall);
        Builder voiceCallControl(VoiceCallControl control);
        Builder voicemail(Voicemail voicemail);
        Builder messaging(Messaging messaging);
        Builder numberInfo(NumberInfo numberInfo);
        MobilePlatform build();
    }
}
```

Unsupported capabilities get NoOp degradation defaults. For critical operations (send SMS, initiate call), NoOp logs WARNING and returns failure result. For query operations (list, get), NoOp returns empty results.

## NoOpMobilePlatform

`@DefaultBean @ApplicationScoped` in mobile-spi. `id()` returns `"none"`. `supports()` returns `false`. All capability accessors return degradation defaults.

## MobilePlatformService

Registry that discovers all `@ApplicationScoped` MobilePlatform beans.

```java
@ApplicationScoped
public class MobilePlatformService {
    private final Map<String, MobilePlatform> platforms;

    MobilePlatformService(List<MobilePlatform> all) { ... }

    MobilePlatform platform(String id);
    boolean supports(String id);
    Set<String> ids();
}
```

## Provider: mobile-ref

In-memory reference implementation with stateful switchboard. Follows chat-ref/bank-ref pattern.

**Components:**
- `MobileBackend` interface — all operations
- `InMemoryMobileBackend` (`@DefaultBean @ApplicationScoped`) — loads seed data from YAML
- `RefMobilePlatform` — delegates to backend
- `MobileRefBeans` — CDI producer

**Switchboard (in-memory call routing):**
- Agents register numbers with the exchange
- Call initiation routes through the switchboard
- Provider-managed calls: seed data contains pre-canned conversation transcripts keyed by instruction pattern
- Platform-managed calls: switchboard invokes ConversationHandler with canned human responses from seed data
- No pickup → voicemail recorded in-memory

**Seed data** (`src/main/resources/seed/`):
- `numbers.yaml` — pre-registered numbers with capabilities
- `conversations.yaml` — canned conversation scripts (instruction pattern → transcript turns)
- `voicemails.yaml` — pre-loaded voicemail messages

**Canned simulation** via `@SimulationEligible` (D13) — the simulation framework provides strategy selection (sequential, key-lookup, random, replay) and corpus seeding. No hand-written simulation code in mobile-ref.

**Supports:** Determined by seed data configuration. Default seed supports all six capabilities.

## Provider: mobile-dial

Dial (getdial.ai) integration.

**Dependencies:** connectors-api (HttpHelper), platform-api (CredentialResolver)

**Authentication:** `credentialResolver.resolve("dial")` → `{"api-key": "sk_live_xxx"}`. Bearer token auth.

**Capability mapping:**

| Capability | Dial API |
|-----------|----------|
| Sms | `POST /messages` (channel: sms) |
| VoiceCall (provider-managed) | `POST /calls` with outboundInstruction |
| VoiceCall (platform-managed) | Self-hosted LLM WebSocket (`wss://`) |
| VoiceCallControl | `POST /calls/:id/stop`, `transferTo` param |
| Voicemail | Provider-specific (may not be supported natively) |
| Messaging | `POST /messages` (channel: whatsapp/imessage/rcs) |
| NumberInfo | `GET /numbers`, `GET /lookup/number` |

**Self-hosted LLM WebSocket protocol (platform-managed calls):**
1. Dial handles audio (TTS/STT on their side)
2. Connectors opens WebSocket with HMAC-SHA256 auth
3. Dial sends `response_required` with full transcript when human finishes speaking
4. ConversationHandler receives text, performs any actions, returns response text
5. Connectors sends `response` message — Dial speaks it via TTS
6. Loop until call ends

**Inbound:** Webhook events (`message.received`, `call.ended`, `call.transcribed`) via WebhookInboundConnector. Webhook signature verification is mandatory — every existing WebhookInboundConnector (Slack, Teams, WhatsApp, Twilio) verifies signatures before processing, using `SigHelper` with constant-time comparison. Dial's self-hosted LLM WebSocket uses HMAC-SHA256; the webhook endpoint likely uses the same mechanism. Implementation must verify Dial's webhook signing docs and implement accordingly. If Dial does not support webhook signing, inbound events via Dial require alternative security: API key in custom header with constant-time comparison, or restrict webhook endpoint to trusted network paths (reverse proxy with IP allowlist).

**Events:** Long-poll (`GET /events/wait`) or SSE (`GET /events/stream`) for real-time event delivery.

**Supports:** Determined by Dial account capabilities and `supports()` implementation.

## Provider: mobile-twilio

Twilio integration. Absorbs existing TwilioSmsConnector and WhatsAppInboundConnector (D5).

**Dependencies:** connectors-api (HttpHelper), platform-api (CredentialResolver), voice-spi (VoicePlatformService for STT/TTS in platform-managed calls)

**Authentication:** `credentialResolver.resolve("twilio")` → `{"account-sid": "xxx", "auth-token": "xxx"}`. Basic auth.

**Capability mapping:**

| Capability | Twilio API |
|-----------|-----------|
| Sms | Twilio Messages API |
| VoiceCall (provider-managed) | TwiML with `<Say>` + Twilio AI (if available) |
| VoiceCall (platform-managed) | Media Streams WebSocket + VoicePlatform STT/TTS |
| VoiceCallControl | Twilio Call resource update (status: completed), `<Dial>` for transfer |
| Voicemail | Twilio Recordings API |
| Messaging | Twilio WhatsApp Business API |
| NumberInfo | Twilio Incoming Phone Numbers + Lookup API |

**Platform-managed voice calls (Media Streams):**
1. Twilio streams bidirectional audio via WebSocket (Media Streams)
2. mobile-twilio receives audio → VoicePlatform STT → text
3. ConversationHandler receives text, returns response text
4. mobile-twilio takes response text → VoicePlatform TTS → audio
5. Audio sent back to Twilio via WebSocket
6. Loop until call ends

This is the conversation layer that Dial handles natively. For Twilio, mobile-twilio builds it from STT/TTS/WebSocket.

**Legacy absorption (D5):**
- Delete `TwilioSmsConnector` from `connectors-api` module (`connectors-api/src/main/java/io/casehub/connectors/twilio/`)
- Delete `WhatsAppConnector` from `connectors-api` module (`connectors-api/src/main/java/io/casehub/connectors/whatsapp/`)
- Delete `TwilioSmsInboundConnector` from `webhook` module (`webhook/src/main/java/io/casehub/connectors/webhook/`)
- Delete `WhatsAppInboundConnector` from `webhook` module (`webhook/src/main/java/io/casehub/connectors/webhook/`)
- Remove CDI producers from `core/ConnectorsBeans.java` and Spring config from `connectors-spring/ConnectorsAutoConfiguration.java`
- Migrate `ConnectorMessagingApi.sendSms()` and `ConnectorMessagingApi.sendWhatsapp()` — these currently route through `ConnectorService.send("twilio-sms", ...)` and `ConnectorService.send("whatsapp", ...)`. Replace with `ConnectorMobileApi` methods routing through `MobilePlatformService` (see §MCP Tools). Delete the SMS/WhatsApp methods from `ConnectorMessagingApi` — email, Slack, and Teams methods remain.
- Preserve notification-bridge `channelType` contract: mobile-twilio returns `"sms"` and `"whatsapp"` for notification delivery compatibility
- Preserve `connectorId` continuity: mobile-twilio's inbound connectors must use `InboundConnectorIds.TWILIO_SMS` (`"twilio-sms-inbound"`) and `InboundConnectorIds.WHATSAPP` (`"whatsapp-inbound"`) — existing `@ObservesAsync InboundMessage` consumers may filter by connectorId
- Reimplement all capabilities in mobile-twilio's SPI

**Inbound:** WebhookInboundConnector implementations fire InboundMessage CDI events (D14). Preserves existing L2 event bus pattern. channelType "sms" and "whatsapp" maintained for backward compatibility.

**Supports:** Determined by Twilio account capabilities.

## VoicePlatform Relationship (D12)

VoicePlatform and MobilePlatform are peer SPIs at different abstraction levels:
- VoicePlatform: codec-level (audio ↔ text conversion)
- MobilePlatform: telephony-level (phone calls and messaging)

The relationship is a provider implementation detail:
- mobile-dial: NO VoicePlatform dependency (Dial handles audio internally)
- mobile-twilio: DEPENDS on voice-spi (needs STT/TTS for platform-managed calls)
- mobile-spi: NO VoicePlatform dependency

Voicemail transcription (in any provider) uses VoicePlatform STT when available.

## Inbound Architecture (D14)

Inbound messages (SMS received, WhatsApp message, call events) flow through the existing L2 event bus:

```
External platform → webhook POST → WebhookInboundConnector (HMAC verify)
    → InboundConnectorService → Event<InboundMessage>.fireAsync()
    → @ObservesAsync consumers
```

MobilePlatform adds structure to OUTBOUND (capability-based send/call). It does not change the inbound event model. Provider modules contribute WebhookInboundConnector implementations that fire InboundMessage events with appropriate channelTypes.

**Connector ID continuity:** mobile-twilio's inbound connectors must reuse the exact `connectorId` strings from the absorbed connectors: `InboundConnectorIds.TWILIO_SMS` (`"twilio-sms-inbound"`) and `InboundConnectorIds.WHATSAPP` (`"whatsapp-inbound"`). Existing `@ObservesAsync InboundMessage` consumers may filter by `connectorId`. Changing these IDs would be a breaking change to downstream consumers.

## MCP Tools

New `ConnectorMobileApi` class in the `graphql` module, following the per-domain API pattern established by `ConnectorMessagingApi` (`connectors/messaging`), `ConnectorVoiceApi` (`connectors/voice`), `ConnectorChatApi` (`connectors/chat`), etc.

```java
@McpDomain(value = "connectors/mobile", app = "connectors",
    basePath = "/api/connectors/mobile",
    summary = "Mobile connector — SMS, voice calls, messaging, voicemail")
@ApplicationScoped
public class ConnectorMobileApi { ... }
```

| Method | Annotation | Capability | Operation |
|--------|-----------|-----------|-----------|
| `sendSms(to, body)` | `@PlatformMutation` | Sms | Send SMS to a number |
| `call(to, instruction)` | `@PlatformMutation` | VoiceCall | Make a voice call (provider-managed) |
| `callAgent(to, agentBrief)` | `@PlatformMutation` | VoiceCall | Make an agentic voice call (platform-managed) |
| `hangUp(callId)` | `@PlatformMutation` | VoiceCallControl | Hang up a call |
| `transfer(callId, transferTo)` | `@PlatformMutation` | VoiceCallControl | Transfer a call |
| `listVoicemails()` | `@PlatformQuery` | Voicemail | List voicemails |
| `transcribeVoicemail(id)` | `@PlatformMutation` | Voicemail | Transcribe a voicemail |
| `sendMessage(to, body, channel)` | `@PlatformMutation` | Messaging | Send WhatsApp/iMessage/RCS |
| `listNumbers()` | `@PlatformQuery` | NumberInfo | List available numbers |
| `lookupNumber(number)` | `@PlatformQuery` | NumberInfo | Check number capabilities |

Methods annotated with `@GeneratePlugin` produce `@Plugin` records for the playbook engine at compile time — no separate `mobile-pipeline-steps` module needed. The `mobile-call` and `mobile-send-sms` playbook steps in the dinner booking playbook are generated from these methods.

## Dinner Booking Playbook

Five domains in one playbook: location + mobile + email + chat.

```yaml
name: Dinner booking — find, call, confirm, notify
steps:
  # 1. Find Thai restaurants nearby
  - location-search-nearby:
      latitude: 51.5074
      longitude: -0.1278
      category: restaurant
      keyword: thai
      radius: 1000
    capture: restaurants

  # 2. Get details of the top result
  - location-place-details:
      placeId: ${restaurants.items[0].placeId}
    capture: restaurant

  # 3. Voice call — agent books a table
  - mobile-call:
      to: ${restaurant.phone}
      instruction: >
        You are calling ${restaurant.name} to book a table for 2
        at 7pm tonight for Smith. Be polite. Confirm the booking
        details before ending the call. If 7pm is unavailable,
        accept any time within 1 hour of 7pm.
    capture: booking

  # 4. Send SMS confirmation to ourselves
  - mobile-send-sms:
      to: "+447700900000"
      body: >
        Booking confirmed at ${restaurant.name}.
        ${booking.transcript}. Bon appetit!
    capture: smsNotification

  # 5. Send email confirmation to restaurant
  - messaging-send-email:
      to: bookings@thai-garden.example.com
      subject: "Booking confirmation — Smith, 2 guests, 7pm tonight"
      body: "This confirms our booking for 2 guests at 7pm tonight. Name: Smith."
    capture: emailConfirmation

  # 6. Chat notification with full details
  - chat-send:
      channel: personal
      message: >
        Dinner tonight sorted. ${restaurant.name} — table for 2
        at 7pm. ${restaurant.address}.
    capture: chatNotification
```

The voice call is a single step — the agent gives the goal, Dial's AI (or the platform's agent via ConversationHandler) handles the conversation. No TTS/STT steps needed.

For the playbook module, the `mobile-call` step uses eidos AgentDescriptor to define the agent brief (D9) — best practice for any LLM invocation.

## Testing Strategy

- **mobile-ref** for all playbook tests — local exchange with canned conversations, no network, no API keys
- **@SimulationEligible** (D13) for capability-level simulation with strategy selection
- **Integration tests** in `integration-tests/` module for real provider tests (gated by API key presence):
  - Dial: real SMS send, real voice call (free $5 credit)
  - Twilio: real SMS send, real voice call (free $15 trial credit)
- **Legacy absorption tests**: verify notification-bridge channelType compatibility after TwilioSmsConnector/WhatsAppInboundConnector removal

## Deferred Scope (GitHub Issues)

- connectors#207 — VoiceCallStreaming capability (v2, from D1)
- connectors#208 — Numbrrs provider mobile-numbrrs (v2, from D6)
- connectors#209 — Number provisioning and management (v2, from D10)

## References

- Dial API docs: docs.getdial.ai
- Dial pricing: $5 free, $3/mo/number, $0.02/SMS, $0.13-0.22/min voice
- Dial self-hosted LLM WebSocket: HMAC-SHA256 auth, response_required/response protocol
- Twilio Media Streams: bidirectional audio WebSocket
- TravelPlatform SPI pattern: inner capabilities, @SimulationEligible, supports(), NoOp, PlatformService
- ChatPlatform Builder pattern: composed capabilities, graceful degradation
- ARC42STORIES.MD §1: library boundary rule
- ADR-0011: @SimulationEligible mandate
- eidos-api AgentDescriptor: best practice for agent briefs (playbooks module)
- VoicePlatform SPI: peer SPI for STT/TTS (mobile-twilio dependency)
- decisions.md: 14 design decisions (D1-D14)
