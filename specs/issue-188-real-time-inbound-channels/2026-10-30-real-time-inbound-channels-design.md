# Real-Time Inbound Channels — Design Spec

**Issue:** casehubio/connectors#188
**Date:** 2026-10-30
**Status:** Draft

## 1. Overview

This epic enhances the connectors inbound infrastructure with three capabilities: WhatsApp voice note media download, Gmail push notifications via Pub/Sub, and a VoicePlatform SPI for speech-to-text and text-to-speech. It also provides a playbook-compatible STT step type for consuming applications to compose into audio processing pipelines.

### 1.1 What This Epic Delivers

| Deliverable | Module | Type |
|-------------|--------|------|
| VoicePlatform SPI (STT + TTS) | `voice-spi` | New |
| Reference VoicePlatform | `voice-ref` | New |
| Gmail push via Pub/Sub | `email-google-inbound` | New |
| STT playbook step type | `voice-pipeline-steps` | New |
| WhatsApp media download | `webhook` | Enhancement |
| Agency SPI adapter | `agency/speech-sherpa` | Cross-repo |

### 1.2 What This Epic Does Not Deliver

- Full speech-sherpa migration from agency (deferred to #187)
- Audio processing capabilities: denoising, VAD, speaker ID, diarization, text cleanup, lip sync (D11 — these are processing capabilities, not channel interfaces; GitHub issue to be created)
- Pipeline playbook YAML (D12 — consuming applications author their own; GitHub issue to be created)
- Gmail pull subscription (D5 — IMAP IDLE already serves that role; GitHub issue to be created)

## 2. Architecture

### 2.1 Dependency Graph

```
voice-pipeline-steps ──→ voice-spi ←── voice-ref
                              ↑
                    agency/speech-sherpa
                    (implements VoicePlatform)

voice-spi ──→ connectors-api (UnsupportedCapabilityException)
voice-spi ──→ simulation-api (@SimulationEligible)

email-google-inbound ──→ email-google ──→ connectors-api

webhook (enhanced) ──→ connectors-api (existing deps unchanged)
```

### 2.2 Cross-Repo Impact

Agency changes (on main):
- `speech-sherpa` POM: add dependency on `casehub-connectors-voice-spi`
- New `SherpaVoicePlatform implements VoicePlatform` adapter class — delegates to existing `SpeechToTextService` and `TextToSpeechService`
- `speech-api` types remain in agency; adapter bridges to connectors SPI types
- Requires IntelliJ workspace spanning both repos

### 2.3 Playbook Integration

Connectors provides a `@Plugin(streaming=true)` step type (`stt-transcribe`) as a CDI bean. Media download is handled inline by each connector. Consuming applications compose step types into playbook YAML with their own routing, retry, and timeout requirements.

The platform's `FusionPass` automatically fuses consecutive type-compatible streaming steps. For this epic's single-step STT pipeline, the playbook engine provides per-step retry/timeout decorators. Future steps (denoise, VAD) can be added and will be automatically fused with STT by the compiler.

## 3. VoicePlatform SPI (`voice-spi`)

### 3.1 Platform Interface

```java
@SimulationEligible(name = "voice-platform",
    capabilities = {"speechToText", "textToSpeech"})
public interface VoicePlatform {
    String id();
    boolean supports(Class<?> capability);

    SpeechToText speechToText();
    TextToSpeech textToSpeech();
}
```

Follows the connectors capability pattern: `ChatPlatform` (10 capabilities), `BankPlatform` (2 capabilities). `VoicePlatform` starts with 2 — additional families added incrementally when a concrete consumer outside agency requires them.

**Capability accessor contract:** `speechToText()` and `textToSpeech()` throw `UnsupportedCapabilityException` when `supports()` returns `false` for the corresponding capability class. This follows the newer platform convention established by `ContactsPlatform`, `CommercePlatform`, `TravelPlatform`, and `HealthPlatform` — NoOp implementations throw with structured error metadata (operation, capability, provider, supportedCapabilities) for LLM self-correction. The caller pattern is: check `supports(SpeechToText.class)` before calling `speechToText()`, or catch `UnsupportedCapabilityException` at the API boundary.

### 3.2 SpeechToText Capability

```java
public interface SpeechToText {
    TranscriptionResult transcribe(byte[] audio, TranscriptionOptions options);
}
```

File-based transcription of downloaded voice notes. Streaming STT (`RecognitionStream`) is an agency-level concern for the avatar WebSocket, not part of the connectors SPI.

### 3.3 TextToSpeech Capability

```java
public interface TextToSpeech {
    SynthesisResult synthesise(String text, SynthesisOptions options);
    default void warmUp() {}
}
```

### 3.4 Model Types

```java
public record TranscriptionOptions(
        String audioFormat,
        String languageHint,
        String modelSize,
        String vocabularyHint) {
    public static TranscriptionOptions defaults() { ... }
    public TranscriptionOptions withLanguageHint(String hint) { ... }
    public TranscriptionOptions withModelSize(String size) { ... }
}
```

**`modelSize` convention:** portable tier values `"small"`, `"medium"`, `"large"`. Providers map tiers to their own model identifiers (e.g. sherpa-onnx maps `"small"` → `whisper-tiny`, `"large"` → `whisper-large-v3`). Provider-specific values (e.g. `"whisper-large-v3"`) are also accepted — the SPI passes them through to the provider unchanged. `String` rather than an enum because model tiers are inherently provider-specific; a shared enum creates a coupling point that must be updated for every new provider.

```java

public record TranscriptionResult(String text, String language, double confidence) {}

public record SynthesisOptions(
        String voice,
        String language,
        String audioFormat,
        boolean includePhonemes) {}

public record SynthesisResult(byte[] audioData, String audioFormat) {}
```

Audio uses `byte[]` matching the existing `Attachment.content()` pattern (heap-resident, V1 constraint documented in `Attachment` javadoc).

### 3.5 Exception Handling

`UnsupportedCapabilityException` (existing in `connectors-api`) thrown from `speechToText()` and `textToSpeech()` when `supports()` returns `false` for the corresponding capability class. Follows the convention established by `ContactsPlatform`, `CommercePlatform`, `TravelPlatform`, and `HealthPlatform`: the NoOp implementation (`NoOpVoicePlatform`) returns `false` for all `supports()` checks and throws `UnsupportedCapabilityException` with structured metadata from every capability accessor. The graphql layer's `requireCapability` helper checks `supports()` before each call and provides recovery metadata for LLM self-correction.

## 4. Reference Implementation (`voice-ref`)

- ID: `"ref"`
- `SpeechToText`: returns `TranscriptionResult("Hello, this is a test transcription.", "en", 0.95)` for any input
- `TextToSpeech`: returns a short silent WAV `byte[]` with correct headers
- `supports()`: returns `true` for both capabilities
- `@SimulationEligible` for simulation framework wrapper generation

Follows the pattern of `chat-ref`, `calendar-ref`, `bank-ref`, `commerce-ref`.

## 5. WhatsApp Media Download (webhook enhancement)

### 5.1 Current State

`WhatsAppInboundConnector.extractContent()` returns the media URL as a string for non-text types. Voice notes arrive as `type=audio` with a media ID but no binary content. The existing code has a "v1 limitation" comment acknowledging this gap.

### 5.2 New Behaviour

**New `WebhookResult` variant:**

```java
record DeferredDelivery(
    List<InboundMessage> immediate,
    Supplier<List<InboundMessage>> deferred) implements WebhookResult {}
```

Added to the sealed `WebhookResult` interface. The router handles it as: deliver `immediate` messages synchronously (as with `Delivered`), then spawn a single virtual thread to execute the `deferred` supplier and deliver its results. This preserves the architectural separation from ADR-0001 — the connector remains passive (returns data), the router owns delivery and threading.

**WhatsApp connector flow:**

1. Webhook arrives → parse messages → split into text and media messages
2. If text-only: return `WebhookResult.Delivered(textMessages)` — existing behaviour, unchanged
3. If media present: return `WebhookResult.DeferredDelivery(textMessages, () -> downloadMedia(mediaMessages))`
4. Router delivers text messages immediately, spawns one virtual thread for the deferred supplier
5. Deferred supplier downloads media sequentially (one at a time, following the Discord pattern):
   - `GET https://graph.facebook.com/v18.0/{media-id}` with Bearer token → JSON with download URL
   - `GET {download-url}` with Bearer token → binary bytes
6. Wrap each in `Attachment(filename, contentType, bytes)` where `filename` is derived from the WhatsApp API response (`filename` field for documents, generated name with correct extension for other types — e.g. `voice-note.ogg` for `audio/ogg`)
7. Return `List<InboundMessage>` from the supplier — the router delivers them via `service.receive()`

**Router changes (`WebhookRouter.dispatch`):**

```java
case WebhookResult.DeferredDelivery(var immediate, var deferred) -> {
    immediate.forEach(service::receive);
    Thread.ofVirtual().name("webhook-deferred-" + id).start(() -> {
        try {
            deferred.get().forEach(service::receive);
        } catch (Exception e) {
            LOG.log(Level.WARNING, "Deferred delivery failed for " + id, e);
        }
    });
    yield Response.ok().build();
}
```

A single webhook POST containing both text and media messages produces temporally separated deliveries: text messages arrive immediately, media messages arrive after download completes. Consumers must not assume a single webhook POST maps to a single delivery call.

Sequential download within the supplier naturally bounds concurrency — one download at a time per webhook POST, matching the Discord attachment download pattern. Meta Graph API rate limits (typically 200 calls/phone-number/hour) are not at risk under normal load.

Uses `HttpHelper.CLIENT` per the shared HTTP client protocol.

### 5.3 Retry Strategy

3 attempts with exponential backoff (1s, 2s, 4s). WhatsApp media URLs expire after a configurable period (default 5 minutes) — immediate download within seconds of webhook receipt is well within the window.

### 5.4 Failure Handling

On final failure, deliver `InboundMessage` with:
- Empty attachments list
- `metadata["download-failed"] = "true"`
- `metadata["media-url"] = url`
- `metadata["media-type"] = type`

Consumer decides corrective action (alert user, queue for manual review).

### 5.5 Scope

All media types get downloaded, not just audio. Eliminates the "v1 limitation". The download mechanism is type-agnostic — voice notes are the motivation but images, documents, video, and stickers all benefit.

## 6. Gmail Push via Pub/Sub (`email-google-inbound`)

### 6.1 How Gmail Push Works

1. App calls `users.watch()` on Gmail API with a Pub/Sub topic and label filter
2. Google publishes notifications to Pub/Sub when matching mail arrives
3. Pub/Sub push subscription posts to our webhook endpoint
4. Notification contains `emailAddress` and `historyId` — not message content
5. App calls `history.list(startHistoryId)` via `GmailHistoryClient` for new message IDs
6. App calls `messages.get()` via `GoogleEmailPlatform` for content

**`GmailHistoryClient`** — new class in the `email-google` module. Gmail's History API (`users.history.list`) is a fundamentally different access pattern from the mailbox-browsing pattern `EmailPlatform` was designed for. It returns `History` resources containing `messagesAdded`, `messagesDeleted`, `labelsAdded`, `labelsRemoved` — a change log, not a message list. This does not belong on the provider-agnostic `EmailPlatform` SPI.

```java
public class GmailHistoryClient {
    private final ServiceConnectionProvider connectionProvider;

    public List<String> getNewMessageIds(String emailAddress, long startHistoryId);
    public long getCurrentHistoryId(String emailAddress);
}
```

Uses the same `ServiceConnectionProvider` as `GoogleEmailPlatform` for per-user OAuth2 tokens.

### 6.2 Connector

```java
public class GmailPushInboundConnector extends WebhookInboundConnector {
    public static final String ID = InboundConnectorIds.GMAIL_PUSH;
    // "gmail-push-inbound"
}
```

- Webhook endpoint: `POST /connectors/gmail-push-inbound/webhook`
- Connector type: `InboundConnectorTypes.EMAIL` (same as IMAP — both produce email `InboundMessage`s; intentional: both deliver email, differentiation available via `connectorId` which is `gmail-push-inbound` vs `email-inbound`, and via the CloudEvent `source` URI which includes the `connectorId`)
- Depends on `email-google` module for `GoogleEmailPlatform` (message fetching) and `GmailHistoryClient` (history-based change tracking)

### 6.3 Authentication

Pub/Sub push subscriptions use OIDC JWT bearer token verification. Google sends a signed JWT in the `Authorization: Bearer` header of each push notification. The connector validates:

1. **Signature** — RS256, verified against Google's OIDC JWKS keys (fetched from `https://www.googleapis.com/oauth2/v3/certs`, cached and refreshed on key rotation)
2. **Issuer** — `https://accounts.google.com`
3. **Audience** — matches `casehub.connectors.gmail.push.pubsub-audience` config
4. **Expiry** — JWT `exp` claim is not past

**Implementation:** uses `smallrye-jwt` for JWT verification with Google's OIDC provider URL configured as the issuer. This is a per-endpoint validation scoped to the Gmail push webhook — not a global Quarkus OIDC tenant — so it uses `JsonWebToken` parsing directly rather than JAX-RS `@RolesAllowed` integration. Key caching and rotation are handled by the SmallRye JWT JWKS client.

No HMAC signature (unlike Slack/WhatsApp webhooks — those use platform-specific shared secret verification).

### 6.4 Watch Lifecycle

`GmailWatchManager` is an `@ApplicationScoped` CDI bean that manages per-user Gmail watch registrations.

**Startup:** observes `@Observes StartupEvent` to register watches for all configured user accounts. For each user:
1. Calls `users.watch()` with the configured Pub/Sub topic and label filter
2. Stores the returned `historyId` as the initial baseline for that user via `HistoryStore`
3. On failure: logs `WARNING`, skips the user, and retries on the next scheduled refresh cycle

**Refresh:** a `@Scheduled` method runs daily (configurable via `casehub.connectors.gmail.push.watch-refresh-interval`). Google watches expire after 7 days; daily refresh ensures watches are renewed well before expiry. The refresh iterates all configured users and re-calls `users.watch()` — Google treats this as idempotent (re-registration returns the existing watch).

**Per-user management:** watches are per-user (Gmail API requirement). The set of monitored users is driven by `ServiceConnectionProvider` — each user with a valid Gmail OAuth2 connection gets a watch. `HistoryStore` is keyed by `emailAddress` so incremental fetching is per-user.

**Connector relationship:** `GmailPushInboundConnector` (the `WebhookInboundConnector`) is injected with `GmailWatchManager` to access `HistoryStore` for history-based message fetching on each push notification. The connector does not manage watches — it only reads history state.

**`HistoryStore` SPI:**
```java
public interface HistoryStore {
    Optional<Long> getLastHistoryId(String emailAddress);
    void setLastHistoryId(String emailAddress, long historyId);
}
```

Default implementation: in-memory `ConcurrentHashMap`. Persistent implementations (database, file) can be provided by consuming applications via CDI alternative.

**Risk: in-memory HistoryStore data loss on restart.** With the default in-memory implementation, all `historyId` state is lost on JVM restart. On restart with no stored `historyId`, the connector fetches the current `historyId` from the Gmail API as a new baseline — no historical messages are replayed. This means messages arriving between shutdown and restart are silently dropped (at-most-once delivery during restarts). This is the inverse of the IMAP IDLE connector's at-least-once risk (§12 Risk #1). Consuming applications requiring reliable delivery across restarts should provide a persistent `HistoryStore` via CDI. A `WARNING` is logged at startup when using the in-memory default.

### 6.5 History Deduplication

`historyId` is monotonically increasing. On notification:
1. Fetch `history.list(startHistoryId=lastSeen)`
2. Process `messagesAdded` events
3. Update `lastSeen` to the response's `historyId`

Duplicate notifications (Pub/Sub at-least-once delivery) produce no duplicate `InboundMessage`s because `history.list` returns only genuinely new messages since `lastSeen`.

### 6.6 Credentials

Uses `ServiceConnectionProvider` (same pattern as `GoogleEmailPlatform`, `GoogleCalendarPlatform`) for per-user OAuth2 tokens. Requires `gmail.readonly` scope (already requested by `email-google`).

### 6.7 Relationship to IMAP IDLE

Gmail push and IMAP IDLE are complementary, not competitive:
- IMAP IDLE: simpler, no GCP infrastructure, requires persistent TCP connection per account
- Gmail push: stateless/webhook-based, requires GCP project (Pub/Sub topic, subscription, IAM), requires public HTTPS endpoint

Both produce `InboundMessage` events with `connectorType = "email"` through the standard CDI async event bus. Consuming applications choose based on deployment constraints.

## 7. STT Step Type (`voice-pipeline-steps`)

### 7.1 Step Definition

```java
@Plugin(value = "stt-transcribe", streaming = true,
        description = "Transcribe audio bytes to text via VoicePlatform STT")
public record SttTranscribePlugin(
        @Optional String language,
        @Optional String model) {

    @Execute
    public TranscriptionResult apply(byte[] audio, VoicePlatform voicePlatform) {
        var options = TranscriptionOptions.defaults()
                .withLanguageHint(language)
                .withModelSize(model);
        return voicePlatform.speechToText().transcribe(audio, options);
    }
}
```

**Compile-time generation:** `@Plugin(streaming = true)` triggers `StepPluginProcessor` at compile time, which generates a `StreamingStep<byte[], TranscriptionResult>` adapter. The record does not implement `StreamingStep` directly — the annotation processor emits the adapter class. In the `@Execute` method signature:
- **First parameter** (`byte[] audio`) — the streaming input type (`I`)
- **Additional parameters** (`VoicePlatform voicePlatform`) — service parameters resolved from `ServiceRegistry` via CDI at `materialize()` time
- **Record components** (`language`, `model`) — configuration parameters populated from playbook YAML

This follows the same pattern as existing streaming plugins (e.g. `StreamingWithConfigPlugin` in the test suite).

### 7.2 Type Contract

- Input: `byte[]` — raw audio bytes from `Attachment.content()`
- Output: `TranscriptionResult` — `text`, `language`, `confidence`
- Service: `VoicePlatform` resolved via `ServiceRegistry` (CDI) at `materialize()` time

### 7.3 Playbook Usage (consuming application)

```yaml
playbook: "1.0"
schema: server

triggers:
  - cloud-event:
      type: io.casehub.connectors.inbound.whatsapp
      filter:
        data.metadata.media-type: audio

steps:
  - stt-transcribe:
      audio: "${event.data.attachments[0].content}"
      language: en
    step: transcript
    timeout: 120s
    retry: { max: 2, delay: 5s, backoff: exponential }

  - rest-request:
      method: POST
      url: /api/messages/enrich
      body:
        messageId: "${event.data.metadata.message-id}"
        transcript: "${result.transcript.output.text}"
        confidence: "${result.transcript.output.confidence}"
    step: enrich
```

### 7.4 Fusion

When future streaming steps are added (e.g. `denoise` producing `byte[]`), the compiler's `FusionPass` automatically fuses `denoise → stt-transcribe` into a single `FusedPipelineStep` because `byte[]` output is assignable to `byte[]` input.

### 7.5 Module Dependencies

- `voice-spi` — `VoicePlatform`, `TranscriptionResult`, `TranscriptionOptions`
- `yaml-plugin-api` — `@Plugin`, `@Execute`, `StreamingStep`

## 8. Agency Cross-Repo Integration

### 8.1 SPI Adapter

New class in `agency/speech-sherpa`:

```java
public class SherpaVoicePlatform implements VoicePlatform {
    private final SpeechToTextService sttService;
    private final TextToSpeechService ttsService;

    @Override
    public SpeechToText speechToText() {
        return (audio, options) -> {
            // Bridge agency TranscriptionOptions → write temp file → delegate to sttService
            // Bridge agency TranscriptionResult → connectors TranscriptionResult
        };
    }

    @Override
    public TextToSpeech textToSpeech() {
        return new TextToSpeech() {
            @Override
            public SynthesisResult synthesise(String text, SynthesisOptions options) {
                // Bridge agency SynthesisOptions → delegate to ttsService
                // Bridge agency SynthesisResult → connectors SynthesisResult
            }
            @Override
            public void warmUp() { ttsService.warmUp(); }
        };
    }
}
```

### 8.2 Migration Tooling

Use IntelliJ refactor-move for the cross-repo migration — move whole modules or packages, not individual classes. This preserves import references, updates POM dependencies, and keeps git history traceable. Requires an IntelliJ workspace open across both `casehubio/connectors` and `casehubio/agency`.

### 8.3 Type Bridging

Agency's `SpeechToTextService.transcribe(Path, TranscriptionOptions)` takes a `Path`; the connectors SPI takes `byte[]`. The adapter writes audio bytes to a temp file, calls the agency service, and maps the result. This indirection is acceptable for voice note transcription (non-streaming, file-based) and disappears when #187 migrates the implementation directly.

### 8.4 POM Changes

`agency/speech-sherpa/pom.xml`:
```xml
<dependency>
    <groupId>io.casehub</groupId>
    <artifactId>casehub-connectors-voice-spi</artifactId>
    <version>${casehub-connectors.version}</version>
</dependency>
```

## 9. Testing Strategy

| Module | Test Approach |
|--------|---------------|
| `voice-spi` | Compile-time only — pure interfaces and records |
| `voice-ref` | Verify canned responses, `supports()` introspection, `@SimulationEligible` wrapper generation |
| `voice-pipeline-steps` | `voice-ref` as provider — `stt-transcribe` step produces `TranscriptionResult` |
| `email-google-inbound` | Mock `GoogleEmailPlatform` for message fetch, test JWT validation, history deduplication, watch lifecycle |
| `webhook` enhancement | Mock Meta Graph API responses, test async download, retry (3 attempts), failure metadata delivery |
| Agency adapter | Verify `SherpaVoicePlatform` delegates to existing speech services, type bridging correctness |

## 10. Constants and Registration

### 10.1 New InboundConnectorIds

```java
public static final String GMAIL_PUSH = "gmail-push-inbound";
```

### 10.2 New MCP Tools

None for this epic. The `stt-transcribe` step type is a playbook step, not an MCP tool.

### 10.3 Configuration Properties

```properties
# WhatsApp media download (existing config, no new properties)
casehub.connectors.whatsapp.api-token=...
casehub.connectors.whatsapp.phone-number-id=...

# Gmail push
casehub.connectors.gmail.push.enabled=true
casehub.connectors.gmail.push.pubsub-audience=https://example.com
casehub.connectors.gmail.push.watch-labels=INBOX
casehub.connectors.gmail.push.watch-refresh-seconds=518400  # 6 days (refresh before 7-day expiry)
```

## References

- `connectors-api/InboundConnector.java` — pull-based inbound SPI
- `connectors-api/WebhookInboundConnector.java` — webhook-based inbound SPI
- `connectors-api/InboundConnectorIds.java` — connector ID constants
- `connectors-api/InboundMessage.java` — inbound message record
- `connectors-api/Attachment.java` — attachment record (heap-resident byte[])
- `webhook/WhatsAppInboundConnector.java` — existing WhatsApp inbound (to be enhanced)
- `chat-discord/DiscordInboundConnector.java` — virtual thread attachment download pattern
- `chat-signal/SignalInboundConnector.java` — inline attachment download pattern
- `email-google/GoogleEmailPlatform.java` — Gmail API integration with ServiceConnectionProvider
- `email-inbound/EmailInboundConnector.java` — IMAP IDLE inbound (complementary to Gmail push)
- `agency/speech-api` — 33 types across 9 families, basis for VoicePlatform SPI
- `agency/speech-sherpa` — sherpa-onnx/whisper.cpp implementation, future migration target (#187)
- `platform/yaml-plugin-api/StreamingStep.java` — streaming step interface
- `platform/yaml-step-runtime/FusionPass.java` — automatic pipe fusion
- `platform#680` — streaming step type issue (landed)
- `connectors#187` — VoicePlatform SPI epic (full migration)
- Protocol: shared-http-client (PP-20260607-9794cb)
- Protocol: inbound-connector-id-constants (PP-20260607-d4ee52)
- Protocol: paginating-client-fail-soft (PP-20260610-83747b)
