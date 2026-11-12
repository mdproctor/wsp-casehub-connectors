# MobilePlatform SPI Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #206 — MobilePlatform SPI — SMS, voice calls, voicemail, WhatsApp on a unified phone number
**Issue group:** #206

**Goal:** Build a capability-based MobilePlatform SPI with six capabilities (Sms, VoiceCall, VoiceCallControl, Voicemail, Messaging, NumberInfo), two voice call modes (provider-managed and platform-managed via ConversationHandler), and three provider modules (ref, dial, twilio).

**Architecture:** ChatPlatform-style Builder composition with graceful degradation. Inner capability interfaces on MobilePlatform. ConversationHandler callback for platform-managed agentic calls. Provider modules implement the SPI — mobile-dial uses Dial's REST + self-hosted LLM WebSocket; mobile-twilio uses Twilio's APIs + Media Streams WebSocket with VoicePlatform STT/TTS.

**Tech Stack:** Java 21, Quarkus 3.32.2, Jackson YAML (seed data), HttpHelper (HTTP clients), WebSocket (Dial self-hosted LLM, Twilio Media Streams)

## Global Constraints

- Java source level 21, run on Java 26 JVM
- Parent POM: `casehub-connectors-parent` version `0.4-SNAPSHOT`
- All cross-project artifacts: `0.2-SNAPSHOT` from GitHub Packages
- jandex-maven-plugin `3.3.1` required in every module
- Build: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn clean install`
- Package: `io.casehub.connectors.mobile` (spi), `io.casehub.connectors.mobile.ref` (ref), `io.casehub.connectors.mobile.dial` (dial), `io.casehub.connectors.mobile.twilio` (twilio)
- E.164 format for all phone numbers
- All model types are Java records unless mutable state is needed
- Tests: plain JUnit 5 + AssertJ, no Quarkus test runner for unit tests

---

## Batch 1: SPI Foundation (mobile-spi)

### Task 1: Model types, capability interfaces, MobilePlatform, Builder, NoOp, PlatformService

**Files:**
- Create: `mobile-spi/pom.xml`
- Create: `mobile-spi/src/main/java/io/casehub/connectors/mobile/model/SendResult.java`
- Create: `mobile-spi/src/main/java/io/casehub/connectors/mobile/model/SmsMessage.java`
- Create: `mobile-spi/src/main/java/io/casehub/connectors/mobile/model/SmsOptions.java`
- Create: `mobile-spi/src/main/java/io/casehub/connectors/mobile/model/Call.java`
- Create: `mobile-spi/src/main/java/io/casehub/connectors/mobile/model/CallOptions.java`
- Create: `mobile-spi/src/main/java/io/casehub/connectors/mobile/model/CallStatus.java`
- Create: `mobile-spi/src/main/java/io/casehub/connectors/mobile/model/TranscriptTurn.java`
- Create: `mobile-spi/src/main/java/io/casehub/connectors/mobile/model/Speaker.java`
- Create: `mobile-spi/src/main/java/io/casehub/connectors/mobile/model/ConversationHandler.java`
- Create: `mobile-spi/src/main/java/io/casehub/connectors/mobile/model/ConversationResponse.java`
- Create: `mobile-spi/src/main/java/io/casehub/connectors/mobile/model/ConversationAction.java`
- Create: `mobile-spi/src/main/java/io/casehub/connectors/mobile/model/VoicemailMessage.java`
- Create: `mobile-spi/src/main/java/io/casehub/connectors/mobile/model/MessagingOptions.java`
- Create: `mobile-spi/src/main/java/io/casehub/connectors/mobile/model/MessagingChannel.java`
- Create: `mobile-spi/src/main/java/io/casehub/connectors/mobile/model/RichMessage.java`
- Create: `mobile-spi/src/main/java/io/casehub/connectors/mobile/model/MediaAttachment.java`
- Create: `mobile-spi/src/main/java/io/casehub/connectors/mobile/model/PhoneNumber.java`
- Create: `mobile-spi/src/main/java/io/casehub/connectors/mobile/model/NumberCapability.java`
- Create: `mobile-spi/src/main/java/io/casehub/connectors/mobile/model/NumberCapabilities.java`
- Create: `mobile-spi/src/main/java/io/casehub/connectors/mobile/model/NumberStatus.java`
- Create: `mobile-spi/src/main/java/io/casehub/connectors/mobile/model/NumberType.java`
- Create: `mobile-spi/src/main/java/io/casehub/connectors/mobile/model/MessageDirection.java`
- Create: `mobile-spi/src/main/java/io/casehub/connectors/mobile/model/MessageStatus.java`
- Create: `mobile-spi/src/main/java/io/casehub/connectors/mobile/spi/MobilePlatform.java`
- Create: `mobile-spi/src/main/java/io/casehub/connectors/mobile/spi/MobilePlatformService.java`
- Create: `mobile-spi/src/main/java/io/casehub/connectors/mobile/spi/NoOpMobilePlatform.java`
- Create: `mobile-spi/src/test/java/io/casehub/connectors/mobile/spi/MobilePlatformBuilderTest.java`
- Create: `mobile-spi/src/test/java/io/casehub/connectors/mobile/spi/NoOpMobilePlatformTest.java`
- Create: `mobile-spi/src/test/java/io/casehub/connectors/mobile/spi/MobilePlatformServiceTest.java`
- Modify: `pom.xml` (parent — add mobile-spi module)

**Interfaces:**
- Produces: `MobilePlatform` interface with inner `Sms`, `VoiceCall`, `VoiceCallControl`, `Voicemail`, `Messaging`, `NumberInfo` capability interfaces. `MobilePlatform.Builder`. `MobilePlatformService` registry. All model records.

- [ ] **Step 1: Create mobile-spi/pom.xml**

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 https://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>
    <parent>
        <groupId>io.casehub</groupId>
        <artifactId>casehub-connectors-parent</artifactId>
        <version>0.4-SNAPSHOT</version>
    </parent>
    <artifactId>casehub-connectors-mobile-spi</artifactId>
    <name>CaseHub Connectors — Mobile SPI</name>

    <dependencies>
        <dependency>
            <groupId>io.casehub</groupId>
            <artifactId>casehub-connectors-api</artifactId>
        </dependency>
        <dependency>
            <groupId>io.casehub</groupId>
            <artifactId>casehub-platform-simulation-api</artifactId>
        </dependency>
        <dependency>
            <groupId>io.quarkus</groupId>
            <artifactId>quarkus-arc</artifactId>
        </dependency>

        <dependency>
            <groupId>io.quarkus</groupId>
            <artifactId>quarkus-junit5</artifactId>
            <scope>test</scope>
        </dependency>
        <dependency>
            <groupId>org.assertj</groupId>
            <artifactId>assertj-core</artifactId>
            <scope>test</scope>
        </dependency>
    </dependencies>

    <build>
        <plugins>
            <plugin>
                <groupId>io.smallrye</groupId>
                <artifactId>jandex-maven-plugin</artifactId>
                <version>3.3.1</version>
                <executions>
                    <execution>
                        <id>make-index</id>
                        <goals><goal>jandex</goal></goals>
                        <phase>process-classes</phase>
                    </execution>
                </executions>
            </plugin>
        </plugins>
    </build>
</project>
```

- [ ] **Step 2: Add mobile-spi to parent pom.xml**

Insert `<module>mobile-spi</module>` in the parent `pom.xml` between `voice-pipeline-steps` and `graphql` (around line 79).

- [ ] **Step 3: Write model records**

Create all model records in `mobile-spi/src/main/java/io/casehub/connectors/mobile/model/`. Key records:

```java
// SendResult.java
package io.casehub.connectors.mobile.model;

public record SendResult(boolean ok, String messageId, String status, String error) {
    public static SendResult success(String messageId) {
        return new SendResult(true, messageId, "sent", null);
    }
    public static SendResult failure(String error) {
        return new SendResult(false, null, "failed", error);
    }
}

// CallStatus.java
public enum CallStatus { QUEUED, RINGING, IN_PROGRESS, COMPLETED, FAILED, BUSY, NO_ANSWER, CANCELLED }

// Speaker.java
public enum Speaker { AGENT, HUMAN, TRANSFER_TARGET }

// TranscriptTurn.java
public record TranscriptTurn(Speaker speaker, String text, long startMs, long endMs) {}

// Call.java
public record Call(String id, String to, String from, CallStatus status,
                   String transcript, List<TranscriptTurn> transcriptTurns,
                   java.time.Duration duration, Instant startedAt, Instant endedAt,
                   String failureReason) {}

// CallOptions.java
public record CallOptions(String fromNumberId, String language, Integer maxDurationSeconds, String transferTo) {
    public static CallOptions defaults() { return new CallOptions(null, null, null, null); }
}

// ConversationAction.java
public sealed interface ConversationAction {
    record Hold() implements ConversationAction {}
    record Transfer(String transferTo) implements ConversationAction {}
    record HangUp() implements ConversationAction {}
}

// ConversationResponse.java
public record ConversationResponse(String text, ConversationAction action) {
    public ConversationResponse(String text) { this(text, null); }
}

// ConversationHandler.java
public interface ConversationHandler {
    ConversationResponse onHumanSpeech(String text, List<TranscriptTurn> fullTranscript);
    default void onCallConnected() {}
    default void onCallEnded(Call completedCall) {}
}

// SmsMessage.java
public record SmsMessage(String id, String from, String to, String body,
                         MessageStatus status, MessageDirection direction, Instant timestamp) {}

// SmsOptions.java
public record SmsOptions(String fromNumberId, List<String> mediaUrls) {}

// MessageDirection.java
public enum MessageDirection { INBOUND, OUTBOUND }

// MessageStatus.java
public enum MessageStatus { SENT, DELIVERED, UNDELIVERED, FAILED, RECEIVED, UNKNOWN }

// VoicemailMessage.java
public record VoicemailMessage(String id, String from, String to, int durationSeconds,
                                Instant timestamp, String audioFormat, String transcription) {}

// MessagingChannel.java
public enum MessagingChannel { WHATSAPP, IMESSAGE, RCS }

// MessagingOptions.java
public record MessagingOptions(MessagingChannel channel, String fromNumberId,
                                List<String> mediaUrls, String templateId) {}

// MediaAttachment.java
public record MediaAttachment(String url, String contentType, long sizeBytes) {}

// RichMessage.java
public record RichMessage(String id, String from, String to, String body,
                           MessagingChannel channel, List<MediaAttachment> media,
                           MessageStatus status, MessageDirection direction, Instant timestamp) {}

// NumberCapability.java
public enum NumberCapability { SMS, VOICE, WHATSAPP, IMESSAGE, RCS }

// NumberStatus.java
public enum NumberStatus { PROVISIONING, READY, FAILED }

// NumberType.java
public enum NumberType { MOBILE, LANDLINE, VOIP }

// PhoneNumber.java
public record PhoneNumber(String id, String number, Set<NumberCapability> capabilities, NumberStatus status) {}

// NumberCapabilities.java
public record NumberCapabilities(String number, String carrier, Set<NumberCapability> capabilities,
                                  String countryCode, NumberType numberType) {}
```

- [ ] **Step 4: Write MobilePlatform interface with capability sub-interfaces**

```java
// MobilePlatform.java
package io.casehub.connectors.mobile.spi;

import io.casehub.connectors.api.Page;
import io.casehub.connectors.api.PageRequest;
import io.casehub.connectors.mobile.model.*;
import io.casehub.platform.simulation.api.SimulationEligible;
import java.util.*;

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

    interface Sms {
        SendResult send(String to, String body);
        SendResult send(String to, String body, SmsOptions options);
        Page<SmsMessage> listMessages(PageRequest page);
    }

    interface VoiceCall {
        Call initiate(String to, String instruction);
        Call initiate(String to, String instruction, CallOptions options);
        Call initiate(String to, ConversationHandler handler);
        Call initiate(String to, ConversationHandler handler, CallOptions options);
        Call getCall(String callId);
        Page<Call> listCalls(PageRequest page);
    }

    interface VoiceCallControl {
        void hangUp(String callId);
        void transfer(String callId, String transferTo);
    }

    interface Voicemail {
        Page<VoicemailMessage> list(PageRequest page);
        VoicemailMessage get(String voicemailId);
        void delete(String voicemailId);
        TranscriptionResult transcribe(String voicemailId);
    }

    interface Messaging {
        SendResult send(String to, String body);
        SendResult send(String to, String body, MessagingOptions options);
        Page<RichMessage> listMessages(PageRequest page);
    }

    interface NumberInfo {
        List<PhoneNumber> listNumbers();
        NumberCapabilities lookup(String phoneNumber);
    }

    static Builder builder(String id) {
        return new DefaultMobilePlatformBuilder(id);
    }

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

Note: `TranscriptionResult` — check if `io.casehub.connectors.voice.model.TranscriptionResult` exists in voice-spi. If so, reuse it from connectors-api or create a mobile-specific one. The implementer should check via `ide_find_class("TranscriptionResult")`.

- [ ] **Step 5: Write DefaultMobilePlatformBuilder (package-private)**

```java
// DefaultMobilePlatformBuilder.java (package-private)
package io.casehub.connectors.mobile.spi;

final class DefaultMobilePlatformBuilder implements MobilePlatform.Builder {
    private final String id;
    private MobilePlatform.Sms sms;
    private MobilePlatform.VoiceCall voiceCall;
    private MobilePlatform.VoiceCallControl voiceCallControl;
    private MobilePlatform.Voicemail voicemail;
    private MobilePlatform.Messaging messaging;
    private MobilePlatform.NumberInfo numberInfo;

    DefaultMobilePlatformBuilder(String id) { this.id = id; }

    // Each setter stores the capability; build() fills nulls with NoOp defaults
    @Override public Builder sms(MobilePlatform.Sms sms) { this.sms = sms; return this; }
    // ... same for all six capabilities

    @Override
    public MobilePlatform build() {
        var supported = new HashSet<Class<?>>();
        if (sms != null) supported.add(MobilePlatform.Sms.class);
        // ... same for all six
        // Fill nulls with NoOp degradation defaults
        if (sms == null) sms = NoOpMobilePlatform.NO_OP_SMS;
        // ... same for all six
        return new BuiltMobilePlatform(id, supported, sms, voiceCall, voiceCallControl,
                                       voicemail, messaging, numberInfo);
    }
}
```

- [ ] **Step 6: Write BuiltMobilePlatform (package-private record)**

```java
// BuiltMobilePlatform.java (package-private)
record BuiltMobilePlatform(String id, Set<Class<?>> supported,
                            Sms sms, VoiceCall voiceCall, VoiceCallControl voiceCallControl,
                            Voicemail voicemail, Messaging messaging, NumberInfo numberInfo)
        implements MobilePlatform {
    @Override public boolean supports(Class<?> capability) { return supported.contains(capability); }
}
```

- [ ] **Step 7: Write NoOpMobilePlatform**

`@DefaultBean @ApplicationScoped`. `id()` = `"none"`. All capability accessors return static NoOp implementations that log WARNING for critical ops and return failure/empty.

```java
@DefaultBean @ApplicationScoped
public class NoOpMobilePlatform implements MobilePlatform {
    private static final Logger LOG = Logger.getLogger(NoOpMobilePlatform.class);

    static final Sms NO_OP_SMS = new Sms() {
        @Override public SendResult send(String to, String body) {
            LOG.warn("NoOp SMS send to " + to + " — no MobilePlatform configured");
            return SendResult.failure("No MobilePlatform configured");
        }
        @Override public SendResult send(String to, String body, SmsOptions options) { return send(to, body); }
        @Override public Page<SmsMessage> listMessages(PageRequest page) { return Page.empty(); }
    };
    // ... same pattern for all six capabilities

    @Override public String id() { return "none"; }
    @Override public boolean supports(Class<?> capability) { return false; }
    @Override public Sms sms() { return NO_OP_SMS; }
    // ... same for all six
}
```

- [ ] **Step 8: Write MobilePlatformService**

```java
@ApplicationScoped
public class MobilePlatformService {
    private final Map<String, MobilePlatform> platforms;

    @Inject
    public MobilePlatformService(Instance<MobilePlatform> all) {
        var map = new LinkedHashMap<String, MobilePlatform>();
        for (var p : all) {
            if (!"none".equals(p.id())) map.put(p.id(), p);
        }
        this.platforms = Map.copyOf(map);
    }

    public MobilePlatform platform(String id) {
        var p = platforms.get(id);
        if (p == null) throw new IllegalArgumentException("Unknown mobile platform: " + id);
        return p;
    }
    public boolean supports(String id) { return platforms.containsKey(id); }
    public Set<String> ids() { return platforms.keySet(); }
}
```

- [ ] **Step 9: Write failing tests**

`MobilePlatformBuilderTest.java`:
- `builderWithSmsOnly_supportsOnlySms` — build with sms(), verify supports(Sms.class) true, supports(VoiceCall.class) false
- `builderWithAllCapabilities_supportsAll` — build with all six, verify all supports() true
- `unsupportedCapability_returnsDegradationDefault` — build with sms() only, call voiceCall().initiate() → verify SendResult/Call failure

`NoOpMobilePlatformTest.java`:
- `id_returnsNone`
- `supports_alwaysFalse`
- `smsSend_returnsFailure`
- `listMessages_returnsEmpty`

`MobilePlatformServiceTest.java`:
- `registersNonNonePlatforms`
- `unknownId_throws`
- `ids_returnsRegisteredIds`

- [ ] **Step 10: Run tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn -pl mobile-spi test`
Expected: All tests pass.

- [ ] **Step 11: Commit**

```bash
git add mobile-spi/ pom.xml
git commit -m "feat(#206): add mobile-spi module — MobilePlatform SPI with 6 capabilities, Builder, NoOp, PlatformService"
```

---

## Batch 2: Reference Implementation (mobile-ref)

### Task 2: MobileBackend, InMemoryMobileBackend, seed data, RefMobilePlatform

**Files:**
- Create: `mobile-ref/pom.xml`
- Create: `mobile-ref/src/main/java/io/casehub/connectors/mobile/ref/MobileBackend.java`
- Create: `mobile-ref/src/main/java/io/casehub/connectors/mobile/ref/InMemoryMobileBackend.java`
- Create: `mobile-ref/src/main/java/io/casehub/connectors/mobile/ref/RefMobilePlatform.java`
- Create: `mobile-ref/src/main/java/io/casehub/connectors/mobile/ref/MobileRefBeans.java`
- Create: `mobile-ref/src/main/java/io/casehub/connectors/mobile/ref/SeedLoader.java`
- Create: `mobile-ref/src/main/resources/seed/numbers.yaml`
- Create: `mobile-ref/src/main/resources/seed/conversations.yaml`
- Create: `mobile-ref/src/main/resources/seed/voicemails.yaml`
- Create: `mobile-ref/src/main/resources/seed/messages.yaml`
- Create: `mobile-ref/src/test/java/io/casehub/connectors/mobile/ref/RefMobilePlatformTest.java`
- Modify: `pom.xml` (parent — add mobile-ref module)

**Interfaces:**
- Consumes: `MobilePlatform`, all model types from mobile-spi
- Produces: `RefMobilePlatform` (CDI `@ApplicationScoped` MobilePlatform implementation)

- [ ] **Step 1: Create mobile-ref/pom.xml**

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 https://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>
    <parent>
        <groupId>io.casehub</groupId>
        <artifactId>casehub-connectors-parent</artifactId>
        <version>0.4-SNAPSHOT</version>
    </parent>
    <artifactId>casehub-connectors-mobile-ref</artifactId>
    <name>CaseHub Connectors — Mobile Reference</name>

    <dependencies>
        <dependency>
            <groupId>io.casehub</groupId>
            <artifactId>casehub-connectors-mobile-spi</artifactId>
        </dependency>
        <dependency>
            <groupId>io.quarkus</groupId>
            <artifactId>quarkus-arc</artifactId>
        </dependency>
        <dependency>
            <groupId>com.fasterxml.jackson.dataformat</groupId>
            <artifactId>jackson-dataformat-yaml</artifactId>
        </dependency>
        <dependency>
            <groupId>io.casehub</groupId>
            <artifactId>casehub-platform-yaml-jackson</artifactId>
        </dependency>
        <dependency>
            <groupId>com.fasterxml.jackson.datatype</groupId>
            <artifactId>jackson-datatype-jsr310</artifactId>
        </dependency>

        <dependency>
            <groupId>io.quarkus</groupId>
            <artifactId>quarkus-junit5</artifactId>
            <scope>test</scope>
        </dependency>
        <dependency>
            <groupId>org.assertj</groupId>
            <artifactId>assertj-core</artifactId>
            <scope>test</scope>
        </dependency>
    </dependencies>

    <build>
        <plugins>
            <plugin>
                <groupId>io.smallrye</groupId>
                <artifactId>jandex-maven-plugin</artifactId>
                <version>3.3.1</version>
                <executions>
                    <execution>
                        <id>make-index</id>
                        <goals><goal>jandex</goal></goals>
                        <phase>process-classes</phase>
                    </execution>
                </executions>
            </plugin>
        </plugins>
    </build>
</project>
```

- [ ] **Step 2: Add mobile-ref to parent pom.xml**

Insert `<module>mobile-ref</module>` after `mobile-spi` in parent POM.

- [ ] **Step 3: Write seed data YAML files**

`numbers.yaml` — 3 pre-registered numbers:
```yaml
- id: pn_ref_001
  number: "+441234567001"
  capabilities: [SMS, VOICE, WHATSAPP]
  status: READY
- id: pn_ref_002
  number: "+441234567002"
  capabilities: [SMS, VOICE]
  status: READY
- id: pn_ref_003
  number: "+441234567003"
  capabilities: [SMS]
  status: READY
```

`conversations.yaml` — canned conversations keyed by instruction pattern:
```yaml
- pattern: "book a table"
  turns:
    - speaker: AGENT
      text: "Hello, I'd like to book a table please."
    - speaker: HUMAN
      text: "Of course! How many guests and what time?"
    - speaker: AGENT
      text: "Table for 2 at 7pm tonight, please. The name is Smith."
    - speaker: HUMAN
      text: "Perfect, table for 2 at 7pm for Smith. That's confirmed. See you tonight!"
    - speaker: AGENT
      text: "Thank you very much. Goodbye!"
- pattern: "default"
  turns:
    - speaker: AGENT
      text: "Hello, how can I help?"
    - speaker: HUMAN
      text: "I'm not sure what you're calling about."
    - speaker: AGENT
      text: "I apologise for the confusion. Goodbye."
```

`voicemails.yaml` — 2 pre-loaded voicemails:
```yaml
- id: vm_001
  from: "+441234567002"
  to: "+441234567001"
  durationSeconds: 15
  timestamp: "2026-11-01T10:30:00Z"
  audioFormat: wav
  transcription: null
- id: vm_002
  from: "+449876543210"
  to: "+441234567001"
  durationSeconds: 8
  timestamp: "2026-11-02T14:15:00Z"
  audioFormat: wav
  transcription: null
```

`messages.yaml` — 3 pre-loaded SMS/WhatsApp messages:
```yaml
- id: msg_001
  from: "+441234567001"
  to: "+449876543210"
  body: "Your appointment is confirmed for tomorrow at 2pm."
  status: DELIVERED
  direction: OUTBOUND
  channel: SMS
  timestamp: "2026-11-01T09:00:00Z"
- id: msg_002
  from: "+449876543210"
  to: "+441234567001"
  body: "Thank you, see you then!"
  status: RECEIVED
  direction: INBOUND
  channel: SMS
  timestamp: "2026-11-01T09:05:00Z"
- id: msg_003
  from: "+441234567001"
  to: "+449876543210"
  body: "Reminder: your booking is in 1 hour."
  status: DELIVERED
  direction: OUTBOUND
  channel: WHATSAPP
  timestamp: "2026-11-02T13:00:00Z"
```

- [ ] **Step 4: Write MobileBackend interface**

```java
package io.casehub.connectors.mobile.ref;

import io.casehub.connectors.api.Page;
import io.casehub.connectors.api.PageRequest;
import io.casehub.connectors.mobile.model.*;
import java.util.List;

public interface MobileBackend {
    // SMS
    SendResult sendSms(String from, String to, String body);
    Page<SmsMessage> listSmsMessages(PageRequest page);

    // Voice calls
    Call initiateCall(String from, String to, String instruction);
    Call initiateCall(String from, String to, ConversationHandler handler);
    Call getCall(String callId);
    Page<Call> listCalls(PageRequest page);

    // Voice call control
    void hangUp(String callId);
    void transfer(String callId, String transferTo);

    // Voicemail
    Page<VoicemailMessage> listVoicemails(PageRequest page);
    VoicemailMessage getVoicemail(String id);
    void deleteVoicemail(String id);

    // Messaging (WhatsApp/iMessage/RCS)
    SendResult sendMessage(String from, String to, String body, MessagingChannel channel);
    Page<RichMessage> listRichMessages(PageRequest page);

    // Numbers
    List<PhoneNumber> listNumbers();
    NumberCapabilities lookupNumber(String number);
}
```

- [ ] **Step 5: Write InMemoryMobileBackend**

`@DefaultBean @ApplicationScoped`. Loads seed data via `SeedLoader` at construction. Stores state in `ConcurrentHashMap` for numbers, messages, calls, voicemails.

Key implementation details:
- `sendSms` creates an `SmsMessage` record, stores it, returns `SendResult.success(id)`
- `initiateCall` with instruction: matches instruction against `conversations.yaml` patterns (substring match), returns canned transcript. Falls back to "default" pattern.
- `initiateCall` with ConversationHandler: iterates canned HUMAN turns from seed data, calls `handler.onHumanSpeech()` for each, collects transcript. Calls `handler.onCallConnected()` at start and `handler.onCallEnded()` at end.
- `hangUp`/`transfer`: find active call by ID, update status
- Voicemail transcription: not handled by backend (delegated to VoicePlatform STT by the platform layer; ref just stores/retrieves)

- [ ] **Step 6: Write SeedLoader**

Reads YAML files from classpath `seed/*.yaml` using Jackson YAML ObjectMapper. Same pattern as travel-ref's SeedLoader.

- [ ] **Step 7: Write RefMobilePlatform**

```java
package io.casehub.connectors.mobile.ref;

public class RefMobilePlatform implements MobilePlatform {
    private final MobilePlatform delegate;

    public RefMobilePlatform(MobileBackend backend) {
        this.delegate = MobilePlatform.builder("ref")
            .sms(new RefSms(backend))
            .voiceCall(new RefVoiceCall(backend))
            .voiceCallControl(new RefVoiceCallControl(backend))
            .voicemail(new RefVoicemail(backend))
            .messaging(new RefMessaging(backend))
            .numberInfo(new RefNumberInfo(backend))
            .build();
    }

    @Override public String id() { return "ref"; }
    @Override public boolean supports(Class<?> c) { return delegate.supports(c); }
    @Override public Sms sms() { return delegate.sms(); }
    // ... delegate all six
}
```

Inner classes `RefSms`, `RefVoiceCall`, etc. delegate to `MobileBackend` methods.

- [ ] **Step 8: Write MobileRefBeans CDI producer**

```java
@ApplicationScoped
public class MobileRefBeans {
    @Produces @ApplicationScoped
    RefMobilePlatform refMobilePlatform(InMemoryMobileBackend backend) {
        return new RefMobilePlatform(backend);
    }
}
```

- [ ] **Step 9: Write failing tests**

`RefMobilePlatformTest.java` — plain JUnit 5:
```java
class RefMobilePlatformTest {
    private RefMobilePlatform platform;

    @BeforeEach
    void setUp() {
        platform = new RefMobilePlatform(new InMemoryMobileBackend());
    }

    @Test void id() { assertThat(platform.id()).isEqualTo("ref"); }
    @Test void supportsAllCapabilities() {
        assertThat(platform.supports(MobilePlatform.Sms.class)).isTrue();
        assertThat(platform.supports(MobilePlatform.VoiceCall.class)).isTrue();
        // ... all six
    }
    @Test void sendSms_returnsSuccess() {
        var result = platform.sms().send("+441234567001", "Hello");
        assertThat(result.ok()).isTrue();
        assertThat(result.messageId()).isNotNull();
    }
    @Test void listSmsMessages_includesPreloaded() {
        var page = platform.sms().listMessages(PageRequest.first(10));
        assertThat(page.items()).isNotEmpty();
    }
    @Test void initiateCall_providerManaged_returnsCannedTranscript() {
        var call = platform.voiceCall().initiate("+441234567002", "Book a table for 2 at 7pm");
        assertThat(call.status()).isEqualTo(CallStatus.COMPLETED);
        assertThat(call.transcriptTurns()).isNotEmpty();
        assertThat(call.transcript()).containsIgnoringCase("table for 2");
    }
    @Test void initiateCall_platformManaged_invokesHandler() {
        var responses = new ArrayList<String>();
        var handler = new ConversationHandler() {
            @Override public ConversationResponse onHumanSpeech(String text, List<TranscriptTurn> t) {
                responses.add(text);
                return new ConversationResponse("OK, got it.");
            }
        };
        var call = platform.voiceCall().initiate("+441234567002", handler);
        assertThat(call.status()).isEqualTo(CallStatus.COMPLETED);
        assertThat(responses).isNotEmpty();
    }
    @Test void listVoicemails_includesPreloaded() {
        var page = platform.voicemail().list(PageRequest.first(10));
        assertThat(page.items()).hasSize(2);
    }
    @Test void listNumbers_includesPreloaded() {
        var numbers = platform.numberInfo().listNumbers();
        assertThat(numbers).hasSize(3);
    }
}
```

- [ ] **Step 10: Implement to pass all tests**

Implement `InMemoryMobileBackend` with seed data loading and in-memory state management.

- [ ] **Step 11: Run tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn -pl mobile-spi,mobile-ref test`
Expected: All tests pass.

- [ ] **Step 12: Commit**

```bash
git add mobile-ref/ pom.xml
git commit -m "feat(#206): add mobile-ref module — in-memory MobilePlatform with switchboard, seed data"
```

---

## Batch 3: MCP Tools + Playbook

### Task 3: ConnectorMobileApi and dinner booking playbook

**Files:**
- Create: `graphql/src/main/java/io/casehub/connectors/graphql/ConnectorMobileApi.java`
- Create: `graphql/src/test/java/io/casehub/connectors/graphql/ConnectorMobileApiTest.java`
- Modify: `graphql/pom.xml` (add mobile-spi dependency)
- Modify: `graphql/src/main/java/io/casehub/connectors/graphql/ConnectorOperationsImpl.java` (add MobilePlatformService to connectors report)
- Modify: `playbooks/pom.xml` (add mobile-ref dependency)
- Modify: `playbooks/src/main/playbooks/multi-domain/08-dinner-booking.playbook.yaml`
- Modify: `playbooks/src/test/playbooks/multi-domain/08-dinner-booking.test.yaml`

**Interfaces:**
- Consumes: `MobilePlatformService`, `MobilePlatform`, all model types from mobile-spi
- Produces: MCP tools `mobile_send_sms`, `mobile_call`, `mobile_hangup`, `mobile_transfer`, `mobile_list_voicemails`, `mobile_transcribe_voicemail`, `mobile_send_message`, `mobile_list_numbers`, `mobile_lookup_number`

- [ ] **Step 1: Add mobile-spi dependency to graphql/pom.xml**

```xml
<dependency>
    <groupId>io.casehub</groupId>
    <artifactId>casehub-connectors-mobile-spi</artifactId>
</dependency>
```

- [ ] **Step 2: Write ConnectorMobileApi**

```java
@McpDomain(value = "connectors/mobile", app = "connectors",
    basePath = "/api/connectors/mobile",
    summary = "Mobile connector — SMS, voice calls, messaging, voicemail")
@ApplicationScoped
public class ConnectorMobileApi {

    private final MobilePlatformService mobilePlatformService;

    @Inject
    public ConnectorMobileApi(MobilePlatformService mobilePlatformService) {
        this.mobilePlatformService = mobilePlatformService;
    }

    @PlatformMutation("Send an SMS message to a phone number")
    @RestPath("/send-sms")
    public SendResult sendSms(
            @QueryParam("platform") @DefaultValue("ref") String platform,
            @QueryParam("to") String to,
            @QueryParam("body") String body) {
        var p = mobilePlatformService.platform(platform);
        requireCapability(p, MobilePlatform.Sms.class, "sendSms");
        return p.sms().send(to, body);
    }

    @PlatformMutation("Make a voice call with an instruction for the AI agent")
    @RestPath("/call")
    public Call call(
            @QueryParam("platform") @DefaultValue("ref") String platform,
            @QueryParam("to") String to,
            @QueryParam("instruction") String instruction) {
        var p = mobilePlatformService.platform(platform);
        requireCapability(p, MobilePlatform.VoiceCall.class, "call");
        return p.voiceCall().initiate(to, instruction);
    }

    @PlatformMutation("Hang up an active call")
    @RestPath("/hangup")
    public void hangUp(
            @QueryParam("platform") @DefaultValue("ref") String platform,
            @QueryParam("callId") String callId) {
        var p = mobilePlatformService.platform(platform);
        requireCapability(p, MobilePlatform.VoiceCallControl.class, "hangUp");
        p.voiceCallControl().hangUp(callId);
    }

    // ... transfer, listVoicemails, transcribeVoicemail, sendMessage, listNumbers, lookupNumber
    // Follow same pattern: resolve platform, requireCapability, delegate

    private void requireCapability(MobilePlatform p, Class<?> cap, String operation) {
        if (!p.supports(cap)) {
            throw UnsupportedCapabilityException.of(operation, cap.getSimpleName(), p.id(),
                supportedCapabilities(p));
        }
    }

    private List<String> supportedCapabilities(MobilePlatform p) {
        var caps = new ArrayList<String>();
        if (p.supports(MobilePlatform.Sms.class)) caps.add("Sms");
        if (p.supports(MobilePlatform.VoiceCall.class)) caps.add("VoiceCall");
        // ... all six
        return caps;
    }
}
```

- [ ] **Step 3: Wire MobilePlatformService into ConnectorOperationsImpl**

Add `MobilePlatformService` to the constructor injection of `ConnectorOperationsImpl`. Add `"mobile"` platform to the `connectorsReport` method output.

- [ ] **Step 4: Write ConnectorMobileApiTest**

Unit test constructing `ConnectorMobileApi` with a `MobilePlatformService` backed by `RefMobilePlatform`. Test each MCP tool method.

- [ ] **Step 5: Rewrite dinner booking playbook**

Replace current TTS/STT steps with `mobile-call` and `mobile-send-sms`:

```yaml
name: Dinner booking — find, call, confirm, notify
steps:
  - location-search-nearby:
      latitude: 51.5074
      longitude: -0.1278
      category: restaurant
      keyword: thai
      radius: 1000
    capture: restaurants

  - location-place-details:
      placeId: ${restaurants.items[0].placeId}
    capture: restaurant

  - mobile-call:
      to: ${restaurant.phone}
      instruction: >
        You are calling ${restaurant.name} to book a table for 2
        at 7pm tonight for Smith. Be polite. Confirm the booking
        details before ending the call. If 7pm is unavailable,
        accept any time within 1 hour of 7pm.
    capture: booking

  - mobile-send-sms:
      to: "+447700900000"
      body: >
        Booking confirmed at ${restaurant.name}.
        ${booking.transcript}. Bon appetit!
    capture: smsNotification

  - messaging-send-email:
      to: bookings@thai-garden.example.com
      subject: "Booking confirmation — Smith, 2 guests, 7pm tonight"
      body: "This confirms our booking for 2 guests at 7pm tonight. Name: Smith."
    capture: emailConfirmation

  - chat-send:
      channel: personal
      message: >
        Dinner tonight sorted. ${restaurant.name} — table for 2
        at 7pm. ${restaurant.address}.
    capture: chatNotification
```

- [ ] **Step 6: Update dinner booking test yaml**

```yaml
name: Dinner booking tests
tests:
  - name: find restaurant, voice-call, SMS, email, chat
    steps:
      - run-playbook: ../../../main/playbooks/multi-domain/08-dinner-booking.playbook.yaml
        capture: result
      - assertThat: result.succeeded
        equals: true
      - assertThat: result.captures.restaurant.name
        not-null: true
      - assertThat: result.captures.booking.status
        equals: COMPLETED
      - assertThat: result.captures.booking.transcript
        not-null: true
      - assertThat: result.captures.smsNotification.ok
        equals: true
      - assertThat: result.captures.chatNotification.ok
        equals: true
```

- [ ] **Step 7: Add mobile-ref dependency to playbooks/pom.xml**

- [ ] **Step 8: Run tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn -pl graphql,playbooks test`
Expected: All tests pass.

- [ ] **Step 9: Commit**

```bash
git add graphql/ playbooks/ pom.xml
git commit -m "feat(#206): add ConnectorMobileApi MCP tools and rewrite dinner booking playbook"
```

---

## Batch 4: Dial Provider (mobile-dial)

### Task 4: DialClient and DialMobilePlatform

**Files:**
- Create: `mobile-dial/pom.xml`
- Create: `mobile-dial/src/main/java/io/casehub/connectors/mobile/dial/DialClient.java`
- Create: `mobile-dial/src/main/java/io/casehub/connectors/mobile/dial/DialMobilePlatform.java`
- Create: `mobile-dial/src/main/java/io/casehub/connectors/mobile/dial/DialWebSocket.java`
- Create: `mobile-dial/src/main/java/io/casehub/connectors/mobile/dial/DialBeans.java`
- Create: `mobile-dial/src/main/java/io/casehub/connectors/mobile/dial/model/DialCall.java`
- Create: `mobile-dial/src/main/java/io/casehub/connectors/mobile/dial/model/DialMessage.java`
- Create: `mobile-dial/src/main/java/io/casehub/connectors/mobile/dial/model/DialNumber.java`
- Create: `mobile-dial/src/test/java/io/casehub/connectors/mobile/dial/DialClientTest.java`
- Create: `mobile-dial/src/test/java/io/casehub/connectors/mobile/dial/DialMobilePlatformTest.java`
- Modify: `pom.xml` (parent — add mobile-dial module)

**Interfaces:**
- Consumes: `MobilePlatform`, `ConversationHandler`, all model types from mobile-spi. `CredentialResolver` from platform-api. `HttpHelper` from connectors-api.
- Produces: `DialMobilePlatform` (`@ApplicationScoped` MobilePlatform for Dial)

- [ ] **Step 1: Create mobile-dial/pom.xml**

Same structure as mobile-ref but dependencies: `mobile-spi`, `connectors-api` (HttpHelper), `platform-api` (CredentialResolver), `quarkus-arc`, `quarkus-websockets-next` (for self-hosted LLM WebSocket).

- [ ] **Step 2: Add mobile-dial to parent pom.xml**

Insert after `mobile-ref`.

- [ ] **Step 3: Write Dial-specific DTOs**

`DialCall`, `DialMessage`, `DialNumber` — Jackson-annotated records matching Dial's JSON response shapes. Used internally by DialClient; mapped to SPI model types at the boundary.

- [ ] **Step 4: Write DialClient**

HTTP client using `HttpHelper.CLIENT` (java.net.http.HttpClient). Methods:

```java
class DialClient {
    private static final String BASE_URL = "https://api.getdial.ai/v1";
    private final String apiKey;

    // SMS/Messaging
    DialMessage sendMessage(String to, String body, String channel, String fromNumberId);
    List<DialMessage> listMessages();

    // Voice calls
    DialCall createCall(String to, String instruction, String fromNumberId, String language, Integer maxDuration);
    DialCall getCall(String callId);
    void stopCall(String callId);

    // Numbers
    List<DialNumber> listNumbers();
    DialNumber lookupNumber(String number);
}
```

Each method: build HttpRequest with Bearer auth, send via HttpHelper.CLIENT, parse JSON response with Jackson.

- [ ] **Step 5: Write DialWebSocket**

Self-hosted LLM WebSocket client for platform-managed calls. Connects to `wss://` endpoint with HMAC-SHA256 auth. Handles message protocol:
- Receives `response_required` → calls `ConversationHandler.onHumanSpeech()`
- Sends `response` with handler's response text
- Handles `call_connected` → calls `handler.onCallConnected()`
- Handles call end → calls `handler.onCallEnded()`

Runs on a virtual thread. Handles ConversationAction sealed types: Hold (re-invoke immediately), Transfer, HangUp.

- [ ] **Step 6: Write DialMobilePlatform**

Uses Builder pattern. Wires DialClient to each capability. Provider-managed calls → DialClient REST. Platform-managed calls → DialWebSocket.

```java
@ApplicationScoped
public class DialMobilePlatform implements MobilePlatform {
    private final MobilePlatform delegate;

    @Inject
    DialMobilePlatform(CredentialResolver credentialResolver) {
        var creds = credentialResolver.resolve("dial");
        var client = new DialClient(creds.get("api-key"));
        this.delegate = MobilePlatform.builder("dial")
            .sms(new DialSms(client))
            .voiceCall(new DialVoiceCall(client))
            .voiceCallControl(new DialVoiceCallControl(client))
            .messaging(new DialMessaging(client))
            .numberInfo(new DialNumberInfo(client))
            .build();
    }
    // delegate all methods
}
```

Note: Voicemail capability not included if Dial doesn't support voicemail natively — `supports(Voicemail.class)` returns false.

- [ ] **Step 7: Write tests**

`DialClientTest` — mock HTTP responses using a test HttpClient or recorded responses. Test each DialClient method's request shape (URL, headers, body) and response parsing.

`DialMobilePlatformTest` — verify Builder wiring, supports() results, capability method routing.

- [ ] **Step 8: Run tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn -pl mobile-spi,mobile-dial test`
Expected: All tests pass.

- [ ] **Step 9: Commit**

```bash
git add mobile-dial/ pom.xml
git commit -m "feat(#206): add mobile-dial module — Dial provider with REST + self-hosted LLM WebSocket"
```

---

## Batch 5: Twilio Provider + Legacy Absorption (mobile-twilio)

### Task 5: TwilioClient, TwilioMobilePlatform, legacy absorption

**Files:**
- Create: `mobile-twilio/pom.xml`
- Create: `mobile-twilio/src/main/java/io/casehub/connectors/mobile/twilio/TwilioClient.java`
- Create: `mobile-twilio/src/main/java/io/casehub/connectors/mobile/twilio/TwilioMobilePlatform.java`
- Create: `mobile-twilio/src/main/java/io/casehub/connectors/mobile/twilio/TwilioMediaStreams.java`
- Create: `mobile-twilio/src/main/java/io/casehub/connectors/mobile/twilio/TwilioSmsBridge.java`
- Create: `mobile-twilio/src/main/java/io/casehub/connectors/mobile/twilio/TwilioWhatsAppBridge.java`
- Create: `mobile-twilio/src/test/java/io/casehub/connectors/mobile/twilio/TwilioMobilePlatformTest.java`
- Delete: `connectors-api/src/main/java/io/casehub/connectors/twilio/` (use `ide_refactor_safe_delete`)
- Delete: `connectors-api/src/main/java/io/casehub/connectors/whatsapp/` (use `ide_refactor_safe_delete`)
- Delete: `webhook/src/main/java/io/casehub/connectors/webhook/TwilioSmsInboundConnector.java` (use `ide_refactor_safe_delete`)
- Delete: `webhook/src/main/java/io/casehub/connectors/webhook/WhatsAppInboundConnector.java` (use `ide_refactor_safe_delete`)
- Modify: `core/src/main/java/io/casehub/connectors/ConnectorsBeans.java` — remove TwilioSmsConnector/WhatsAppConnector producers
- Modify: `connectors-spring/src/main/java/.../ConnectorsAutoConfiguration.java` — remove Twilio/WhatsApp auto-config
- Modify: `graphql/src/main/java/.../ConnectorMessagingApi.java` — remove sendSms/sendWhatsapp methods
- Modify: `pom.xml` (parent — add mobile-twilio module)

**Interfaces:**
- Consumes: `MobilePlatform`, model types from mobile-spi, `CredentialResolver`, `HttpHelper`, `VoicePlatformService` from voice-spi
- Produces: `TwilioMobilePlatform` (`@ApplicationScoped` MobilePlatform for Twilio), `TwilioSmsBridge`/`TwilioWhatsAppBridge` (WebhookInboundConnector implementations)

- [ ] **Step 1: Create mobile-twilio/pom.xml**

Dependencies: `mobile-spi`, `connectors-api`, `platform-api`, `voice-spi`, `core` (WebhookInboundConnector), `quarkus-arc`, `quarkus-websockets-next`.

- [ ] **Step 2: Add mobile-twilio to parent pom.xml**

Insert after `mobile-dial`.

- [ ] **Step 3: Write TwilioClient**

HTTP client for Twilio REST API. Basic auth with account-sid:auth-token. Methods for Messages API, Calls API, Recordings API, Incoming Phone Numbers API, Lookup API.

- [ ] **Step 4: Write TwilioMediaStreams**

WebSocket client for Twilio Media Streams. Receives bidirectional audio. Integrates with VoicePlatformService:
- Receive audio from Twilio → `speechToText.transcribe(audio)` → text
- ConversationHandler receives text, returns response text
- Response text → `textToSpeech.synthesise(text)` → audio
- Send audio back to Twilio

Fail-fast check at initiate() time: if VoicePlatformService has no real STT/TTS provider, throw UnsupportedCapabilityException.

- [ ] **Step 5: Write TwilioMobilePlatform**

Builder pattern. Wires TwilioClient to capabilities. Platform-managed voice calls use TwilioMediaStreams.

- [ ] **Step 6: Write inbound connectors (TwilioSmsBridge, TwilioWhatsAppBridge)**

Implement `WebhookInboundConnector`. Fire `InboundMessage` CDI events. Preserve connector IDs: `InboundConnectorIds.TWILIO_SMS` and `InboundConnectorIds.WHATSAPP`.

- [ ] **Step 7: Delete legacy connectors**

Use `ide_refactor_safe_delete` for each file. Then manually clean up producers in `ConnectorsBeans.java`, auto-config in `ConnectorsAutoConfiguration.java`, and MCP methods in `ConnectorMessagingApi.java`.

Check all 25 TwilioSmsConnector references and 19 WhatsAppConnector references via `ide_find_references` before deleting.

- [ ] **Step 8: Write tests**

`TwilioMobilePlatformTest` — verify Builder wiring, supports() results, capability method routing with mocked TwilioClient.

- [ ] **Step 9: Run full build**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn clean install`
Expected: Full build passes. No compilation errors from legacy connector removal.

- [ ] **Step 10: Commit**

```bash
git add mobile-twilio/ connectors-api/ webhook/ core/ connectors-spring/ graphql/ pom.xml
git commit -m "feat(#206): add mobile-twilio module, absorb TwilioSmsConnector and WhatsAppInboundConnector"
```

---

## Batch 6: Integration Tests + Documentation

### Task 6: Integration tests and documentation updates

**Files:**
- Create: `integration-tests/src/test/java/io/casehub/connectors/mobile/MobileRefIntegrationTest.java`
- Modify: `CLAUDE.md` — add mobile modules to module table
- Modify: `docs/guides/consumer-guide.md` — add MobilePlatform section
- Modify: `docs/guides/contributor-guide.md` — add MobilePlatform architecture

**Interfaces:**
- Consumes: All mobile modules

- [ ] **Step 1: Write MobileRefIntegrationTest**

Quarkus `@QuarkusTest` in integration-tests module. Verifies full CDI wiring:
- `@Inject MobilePlatformService` — verify "ref" is registered
- SMS round-trip: send → list → verify
- Voice call: initiate with instruction → verify transcript
- Voice call: initiate with ConversationHandler → verify handler invoked
- Voicemail: list → get → verify
- NumberInfo: listNumbers → verify pre-loaded data

- [ ] **Step 2: Update CLAUDE.md module table**

Add mobile-spi, mobile-ref, mobile-dial, mobile-twilio to the module table. Add mobile MCP tools to the tool listing.

- [ ] **Step 3: Update consumer guide**

Add MobilePlatform quick start section: dependency, inject MobilePlatformService, send SMS, make voice call, check number capabilities.

- [ ] **Step 4: Update contributor guide**

Add MobilePlatform architecture: capability sub-interfaces, ConversationHandler, provider implementation details.

- [ ] **Step 5: Run full build**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn clean install`
Expected: Full build passes.

- [ ] **Step 6: Commit**

```bash
git add integration-tests/ CLAUDE.md docs/
git commit -m "feat(#206): add mobile integration tests and documentation"
```

---

## References

- [2026-11-12-mobile-platform-spi-design.md] — design spec this plan implements
- [decisions.md] — 14 design decisions (D1-D14)
- [travel-spi/] — SPI module pattern (pom.xml, interface structure, @SimulationEligible)
- [travel-ref/] — ref module pattern (Backend, InMemory, SeedLoader, CDI producer)
- [graphql/ConnectorVoiceApi.java] — @McpDomain API pattern
- [graphql/ConnectorOperationsImpl.java] — platform service wiring
- [playbooks/ConnectorPlaybookSuiteTest.java] — YAML test suite pattern
- [GitHub #206] — focal issue
