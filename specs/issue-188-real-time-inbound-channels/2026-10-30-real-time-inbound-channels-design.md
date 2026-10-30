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
- Audio processing capabilities: denoising, VAD, speaker ID, diarization, text cleanup, lip sync (D11 — these are processing capabilities, not channel interfaces)
- Pipeline playbook YAML (D12 — consuming applications author their own)
- Gmail pull subscription (D5 — IMAP IDLE already serves that role)

## 2. Architecture

### 2.1 Dependency Graph

```
voice-pipeline-steps ──→ voice-spi ←── voice-ref
                              ↑
                    agency/speech-sherpa
                    (implements VoicePlatform)

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
@SimulationEligible
public interface VoicePlatform {
    String id();
    boolean supports(Class<?> capability);

    SpeechToText speechToText();
    TextToSpeech textToSpeech();
}
```

Follows the connectors capability pattern: `ChatPlatform` (10 capabilities), `BankPlatform` (2 capabilities). `VoicePlatform` starts with 2 — additional families added incrementally when a concrete consumer outside agency requires them.

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

`UnsupportedCapabilityException` (existing in `connectors-api`) thrown when `supports()` returns false and the caller invokes the capability. Follows the established pattern with structured error metadata.

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

For all media types (audio, image, document, video, sticker), spawn a virtual thread to download the binary via the Meta Graph API:

1. Webhook arrives → parse messages → split into text and media messages
2. Text messages: return immediately as `WebhookResult.Delivered(textMessages)` — existing behaviour, unchanged
3. Media messages: return `WebhookResult.Ignored` (so the router does not deliver them), then spawn a virtual thread per media message
4. Virtual thread: `GET https://graph.facebook.com/v18.0/{media-id}` with Bearer token → JSON with download URL
5. Virtual thread: `GET {download-url}` with Bearer token → binary bytes
6. Wrap in `Attachment(mediaId, contentType, audioBytes)`
7. Deliver `InboundMessage` to sink via `sink.receive()` directly (bypassing the webhook router's `Delivered` path)

A single webhook POST containing both text and media messages produces temporally separated sink deliveries: text messages arrive immediately, media messages arrive after download completes. Consumers must not assume a single webhook POST maps to a single sink call.

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
5. App calls `history.list(startHistoryId)` via `GoogleEmailPlatform` for new message IDs
6. App calls `messages.get()` via `GoogleEmailPlatform` for content

### 6.2 Connector

```java
public class GmailPushInboundConnector extends WebhookInboundConnector {
    public static final String ID = InboundConnectorIds.GMAIL_PUSH;
    // "gmail-push-inbound"
}
```

- Webhook endpoint: `POST /connectors/gmail-push-inbound/webhook`
- Connector type: `InboundConnectorTypes.EMAIL` (same as IMAP — both produce email `InboundMessage`s)
- Depends on `email-google` module for `GoogleEmailPlatform` message fetching

### 6.3 Authentication

Pub/Sub push subscriptions use JWT bearer token verification via Google's OAuth2 certs. The connector validates the JWT `audience` claim matches the configured value. No HMAC signature (unlike Slack/WhatsApp webhooks).

### 6.4 Watch Lifecycle

`GmailWatchManager` CDI bean:
- Calls `users.watch()` at startup
- Refreshes before expiry (Google watches expire after 7 days)
- Watches scoped to configurable labels (e.g. `INBOX` only)
- Stores `historyId` per user for incremental history fetching via a `HistoryStore` SPI

**`HistoryStore` SPI:**
```java
public interface HistoryStore {
    Optional<Long> getLastHistoryId(String emailAddress);
    void setLastHistoryId(String emailAddress, long historyId);
}
```

Default implementation: in-memory `ConcurrentHashMap`. On startup with no stored `historyId`, the connector performs a full sync (fetches current `historyId` from Gmail API as the baseline — no historical messages replayed). Persistent implementations (database, file) can be provided by consuming applications via CDI.

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

### 7.2 Type Contract

- Input: `byte[]` — raw audio bytes from `Attachment.content()`
- Output: `TranscriptionResult` — `text`, `language`, `confidence`
- Service: `VoicePlatform` resolved via `ServiceRegistry` (CDI)

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
