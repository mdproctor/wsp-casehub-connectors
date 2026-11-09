# Voice Playbook Integration Tests — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #188 — Real-time inbound channels
**Issue group:** #188

**Goal:** End-to-end YAML playbook integration tests demonstrating voice + multi-platform scenarios (banking, chat, email) with zero Java test code — pure YAML playbooks, simulation files, and test files.

**Architecture:** ConnectorTestEngine extends YamlTestEngine to pre-register connector reference platforms (voice, bank, chat, email) and domain plugins into the service registry. Each connector domain has one plugin class using Walker's combined-key resolution (`bank balance: "acc-123"` → plugin="bank", operation="balance"). Playbook YAMLs define scenarios, paired simulation YAMLs provide deterministic responses, test YAMLs execute and assert.

**Tech Stack:** Java 21, JUnit Platform TestEngine, platform yaml-test + simulation-core, connectors reference implementations

## Global Constraints

- Java 21 source, Java 26 JVM
- All connector artifacts: version `${project.version}` (0.4-SNAPSHOT)
- Platform artifacts: parent-managed versions (no explicit version tags)
- CDI-free instantiation of reference platforms (no Quarkus test runtime)
- Combined-key YAML syntax for domain steps: `domain operation: "value"`

---

## Batch 1: Engine + Voice Echo

Validates the entire infrastructure stack: module → test engine → domain plugin → playbook → simulation → test YAML. After this batch, `mvn test` in the module runs a pure YAML voice round-trip test.

### Task 1: Module scaffold and ConnectorTestEngine

**Files:**
- Create: `connector-playbook-tests/pom.xml`
- Modify: `pom.xml` (parent — add module)
- Create: `connector-playbook-tests/src/main/java/io/casehub/connectors/playbook/ConnectorTestEngine.java`
- Create: `connector-playbook-tests/src/main/resources/META-INF/services/org.junit.platform.engine.TestEngine`
- Create: `connector-playbook-tests/src/main/java/io/casehub/connectors/playbook/plugins/VoicePlugin.java`
- Create: `connector-playbook-tests/src/main/resources/playbooks/voice-interaction-patterns/01-voice-echo.playbook.yaml`
- Create: `connector-playbook-tests/src/main/resources/playbooks/voice-interaction-patterns/01-voice-echo.simulation.yaml`
- Create: `connector-playbook-tests/src/test/resources/voice-interaction-patterns/01-voice-echo.test.yaml`

**Interfaces:**
- Consumes: `VoicePlatform`, `ReferenceVoicePlatform`, `SpeechToText`, `TextToSpeech`, `TranscriptionResult`, `SynthesisResult` from voice-spi/voice-ref
- Consumes: `YamlTestEngine`, `MapServiceRegistry`, `CompositePluginRegistry`, `SimulationRuntime`, `Definition`, `Result` from platform yaml-test
- Produces: `ConnectorTestEngine` (JUnit Platform engine), `VoicePlugin` (combined-key domain plugin)

- [ ] **Step 1: Create module POM**

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

  <artifactId>casehub-connectors-playbook-tests</artifactId>
  <name>CaseHub Connectors — Playbook Tests</name>
  <description>YAML playbook integration tests for connector platforms.
ConnectorTestEngine extends YamlTestEngine with pre-registered connector
reference platforms and domain plugins for combined-key YAML steps.</description>

  <dependencies>
    <!-- Platform test engine and simulation -->
    <dependency>
      <groupId>io.casehub</groupId>
      <artifactId>casehub-platform-yaml-test</artifactId>
    </dependency>
    <dependency>
      <groupId>io.casehub</groupId>
      <artifactId>casehub-platform-yaml-step-runtime</artifactId>
    </dependency>
    <dependency>
      <groupId>io.casehub</groupId>
      <artifactId>casehub-platform-yaml-plugin-api</artifactId>
    </dependency>
    <dependency>
      <groupId>io.casehub</groupId>
      <artifactId>casehub-platform-yaml-core</artifactId>
    </dependency>
    <dependency>
      <groupId>io.casehub</groupId>
      <artifactId>casehub-platform-simulation-core</artifactId>
    </dependency>
    <dependency>
      <groupId>io.casehub</groupId>
      <artifactId>casehub-platform-simulation-api</artifactId>
    </dependency>

    <!-- Connector SPIs -->
    <dependency>
      <groupId>io.casehub</groupId>
      <artifactId>casehub-connectors-voice-spi</artifactId>
      <version>${project.version}</version>
    </dependency>
    <dependency>
      <groupId>io.casehub</groupId>
      <artifactId>casehub-connectors-voice-ref</artifactId>
      <version>${project.version}</version>
    </dependency>
    <dependency>
      <groupId>io.casehub</groupId>
      <artifactId>casehub-connectors-bank-spi</artifactId>
      <version>${project.version}</version>
    </dependency>
    <dependency>
      <groupId>io.casehub</groupId>
      <artifactId>casehub-connectors-bank-ref</artifactId>
      <version>${project.version}</version>
    </dependency>
    <dependency>
      <groupId>io.casehub</groupId>
      <artifactId>casehub-connectors-chat-spi</artifactId>
      <version>${project.version}</version>
    </dependency>
    <dependency>
      <groupId>io.casehub</groupId>
      <artifactId>casehub-connectors-chat-ref</artifactId>
      <version>${project.version}</version>
    </dependency>
    <dependency>
      <groupId>io.casehub</groupId>
      <artifactId>casehub-connectors-email-spi</artifactId>
      <version>${project.version}</version>
    </dependency>
    <dependency>
      <groupId>io.casehub</groupId>
      <artifactId>casehub-connectors-email-ref</artifactId>
      <version>${project.version}</version>
    </dependency>
    <dependency>
      <groupId>io.casehub</groupId>
      <artifactId>casehub-connectors-api</artifactId>
      <version>${project.version}</version>
    </dependency>

    <!-- Testing -->
    <dependency>
      <groupId>org.junit.jupiter</groupId>
      <artifactId>junit-jupiter</artifactId>
      <scope>test</scope>
    </dependency>
    <dependency>
      <groupId>org.junit.platform</groupId>
      <artifactId>junit-platform-launcher</artifactId>
      <scope>test</scope>
    </dependency>
    <dependency>
      <groupId>org.assertj</groupId>
      <artifactId>assertj-core</artifactId>
      <scope>test</scope>
    </dependency>
  </dependencies>

</project>
```

- [ ] **Step 2: Add module to parent POM**

In `pom.xml`, add `<module>connector-playbook-tests</module>` after `<module>integration-tests</module>` in the `<modules>` section.

- [ ] **Step 3: Write ConnectorTestEngine**

This engine overrides YamlTestEngine's execution to pre-register connector platforms and domain plugins. Study YamlTestEngine.execute() in platform — replicate its structure but add connector-specific registrations.

```java
package io.casehub.connectors.playbook;

import io.casehub.connectors.bank.ref.InMemoryBankBackend;
import io.casehub.connectors.bank.ref.RefBankPlatform;
import io.casehub.connectors.bank.spi.BankPlatform;
import io.casehub.connectors.chat.ref.InMemoryChatBackend;
import io.casehub.connectors.chat.ref.RefChatPlatform;
import io.casehub.connectors.chat.spi.ChatPlatform;
import io.casehub.connectors.email.ref.InMemoryEmailBackend;
import io.casehub.connectors.email.ref.RefEmailPlatform;
import io.casehub.connectors.email.spi.EmailPlatform;
import io.casehub.connectors.playbook.plugins.VoicePlugin;
import io.casehub.connectors.voice.ref.ReferenceVoicePlatform;
import io.casehub.connectors.voice.spi.VoicePlatform;
// Platform yaml-test imports — verify exact class names during implementation
import io.casehub.platform.yaml.test.YamlTestEngine;

public class ConnectorTestEngine extends YamlTestEngine {

    @Override
    public String getId() {
        return "connector-playbook";
    }

    // Override execute() to register connector platforms and domain plugins
    // into the MapServiceRegistry before creating the YamlTestExecutor.
    //
    // Platforms to register:
    //   serviceRegistry.register(VoicePlatform.class, new ReferenceVoicePlatform());
    //   serviceRegistry.register(BankPlatform.class, new RefBankPlatform(new InMemoryBankBackend()));
    //   serviceRegistry.register(ChatPlatform.class, new RefChatPlatform(new InMemoryChatBackend()));
    //   serviceRegistry.register(EmailPlatform.class, new RefEmailPlatform(new InMemoryEmailBackend()));
    //
    // Domain plugins to register:
    //   pluginRegistry.register(VoicePlugin.definition(serviceRegistry));
    //   pluginRegistry.register(BankPlugin.definition(serviceRegistry));
    //   pluginRegistry.register(ChatPlugin.definition(serviceRegistry));
    //   pluginRegistry.register(EmailPlugin.definition(serviceRegistry));
    //
    // Exact override structure depends on YamlTestEngine's execute() method.
    // Read the source during implementation and replicate the logic,
    // inserting the registrations before the executor is created.
}
```

The exact override shape depends on YamlTestEngine's internal structure. During implementation:
1. Read `YamlTestEngine.execute()` in platform
2. Identify where `MapServiceRegistry` and `CompositePluginRegistry` are created
3. Override to add connector registrations after the base setup

- [ ] **Step 4: Write VoicePlugin**

Domain plugin for voice operations. Registered as `"voice"` — Walker combined-key splits `voice transcribe: data` into plugin="voice", operation="transcribe", value=data.

```java
package io.casehub.connectors.playbook.plugins;

import java.util.Map;

import io.casehub.connectors.voice.model.SynthesisOptions;
import io.casehub.connectors.voice.model.SynthesisResult;
import io.casehub.connectors.voice.model.TranscriptionOptions;
import io.casehub.connectors.voice.model.TranscriptionResult;
import io.casehub.connectors.voice.spi.VoicePlatform;
// Platform Definition and Result imports — verify exact classes during implementation
import io.casehub.platform.yaml.plugin.Definition;
import io.casehub.platform.yaml.plugin.Result;

public class VoicePlugin {

    public static Definition definition(VoicePlatform voicePlatform) {
        return Definition.of("voice")
            .execute((inputs) -> {
                String operation = primaryInput(inputs);
                return switch (operation) {
                    case "transcribe" -> transcribe(inputs, voicePlatform);
                    case "synthesise" -> synthesise(inputs, voicePlatform);
                    default -> Result.failed("Unknown voice operation: " + operation);
                };
            })
            .build();
    }

    private static Result transcribe(Map<String, Object> inputs, VoicePlatform platform) {
        byte[] audio = resolveAudio(inputs);
        String language = (String) inputs.getOrDefault("language", "en");
        var options = new TranscriptionOptions(null, language, null, null);
        TranscriptionResult result = platform.speechToText().transcribe(audio, options);
        return Result.of(Map.of(
            "text", result.text(),
            "language", result.language(),
            "confidence", result.confidence()
        ));
    }

    private static Result synthesise(Map<String, Object> inputs, VoicePlatform platform) {
        String text = (String) inputs.get("_value");
        String voice = (String) inputs.get("voice");
        String language = (String) inputs.getOrDefault("language", "en");
        var options = new SynthesisOptions(voice, language, "wav", false);
        SynthesisResult result = platform.textToSpeech().synthesise(text, options);
        return Result.of(Map.of(
            "audioData", result.audioData(),
            "audioFormat", result.audioFormat()
        ));
    }

    private static String primaryInput(Map<String, Object> inputs) {
        // Walker sets the combined-key suffix as the primaryInput.
        // The exact key name depends on the Definition's input declarations.
        // Verify against Walker.resolve() during implementation.
        return (String) inputs.getOrDefault("_subject", inputs.get("operation"));
    }

    private static byte[] resolveAudio(Map<String, Object> inputs) {
        Object value = inputs.get("_value");
        if (value instanceof byte[] bytes) return bytes;
        if (value instanceof String s) return s.getBytes();
        return new byte[0];
    }
}
```

- [ ] **Step 5: Write 01-voice-echo.playbook.yaml**

```yaml
name: Voice echo
description: Transcribe speech then synthesise the transcript back
steps:
  - voice transcribe: "Hello, this is a test."
    language: en
    capture: transcript

  - voice synthesise: "{{transcript.text}}"
    capture: audio
```

- [ ] **Step 6: Write 01-voice-echo.simulation.yaml**

```yaml
methods:
  voice.transcribe:
    strategy: sequential
    corpus:
      - output:
          text: "Hello, this is a test."
          language: en
          confidence: 0.95
  voice.synthesise:
    strategy: sequential
    corpus:
      - output:
          audioData: []
          audioFormat: wav
```

- [ ] **Step 7: Write 01-voice-echo.test.yaml**

```yaml
name: Voice echo tests
tests:
  - name: "transcribe then synthesise round-trip"
    steps:
      - run-playbook: playbooks/voice-interaction-patterns/01-voice-echo.playbook.yaml
        capture: result
      - assertThat: result.succeeded
        equals: true
      - assertThat: result.stepCount
        equals: 2
```

- [ ] **Step 8: Write META-INF/services file**

```
io.casehub.connectors.playbook.ConnectorTestEngine
```

- [ ] **Step 9: Build and verify test passes**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn -pl connector-playbook-tests test -f /Users/mdproctor/claude/casehub/connectors/pom.xml`

Expected: 1 test discovered by ConnectorTestEngine, PASS. The test YAML executes the playbook via the engine, VoicePlugin resolves `voice transcribe:` and `voice synthesise:` steps, ReferenceVoicePlatform provides canned responses.

- [ ] **Step 10: Commit**

```bash
git add connector-playbook-tests/ pom.xml
git commit -m "feat(#188): add connector-playbook-tests module — ConnectorTestEngine + VoicePlugin + voice echo playbook"
```

---

## Batch 2: Agent Sessions + Bank Domain

After this batch, playbooks 02 and 03 pass — showing voice + LLM agent and voice + LLM + bank API integration.

### Task 2: Voice agent playbook (LLM simulation)

**Files:**
- Create: `connector-playbook-tests/src/main/resources/playbooks/voice-interaction-patterns/02-voice-agent.playbook.yaml`
- Create: `connector-playbook-tests/src/main/resources/playbooks/voice-interaction-patterns/02-voice-agent.simulation.yaml`
- Create: `connector-playbook-tests/src/test/resources/voice-interaction-patterns/02-voice-agent.test.yaml`

**Interfaces:**
- Consumes: ConnectorTestEngine (from Task 1), VoicePlugin (from Task 1), AgentSessionPlugins (from platform — `agent-session-open`, `agent-query`, `agent-session-close`)
- Produces: working voice + agent playbook validated by YAML test

- [ ] **Step 1: Write 02-voice-agent.playbook.yaml**

```yaml
name: Voice agent
description: Voice input to agent classification to voice output
steps:
  - voice transcribe: "I need to check my account balance"
    language: en
    capture: transcript

  - agent-session-open classifier:
      system: You are a request classifier. Respond with exactly one word — BALANCE, TRANSFER, or SUPPORT.

  - agent-query classifier: "{{transcript.text}}"
    capture: classification

  - agent-session-close: classifier

  - voice synthesise: "Your request has been classified as {{classification.text}}"
    capture: audio
```

- [ ] **Step 2: Write 02-voice-agent.simulation.yaml**

```yaml
methods:
  voice.transcribe:
    strategy: sequential
    corpus:
      - output:
          text: "I need to check my account balance"
          language: en
          confidence: 0.97
  voice.synthesise:
    strategy: sequential
    corpus:
      - output:
          audioData: []
          audioFormat: wav
```

- [ ] **Step 3: Write 02-voice-agent.test.yaml**

```yaml
name: Voice agent tests
simulate:
  agent:
    - prompt: "balance"
      response: "BALANCE"
tests:
  - name: "voice to agent classification round-trip"
    steps:
      - run-playbook: playbooks/voice-interaction-patterns/02-voice-agent.playbook.yaml
        capture: result
      - assertThat: result.succeeded
        equals: true
      - assertThat: result.stepCount
        equals: 5
```

- [ ] **Step 4: Run test and verify**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn -pl connector-playbook-tests test -f /Users/mdproctor/claude/casehub/connectors/pom.xml`

Expected: 2 test suites pass (01-voice-echo + 02-voice-agent).

- [ ] **Step 5: Commit**

```bash
git add connector-playbook-tests/
git commit -m "feat(#188): add voice-agent playbook — voice + LLM agent classification"
```

### Task 3: BankPlugin + banking assistant playbook

**Files:**
- Create: `connector-playbook-tests/src/main/java/io/casehub/connectors/playbook/plugins/BankPlugin.java`
- Modify: `connector-playbook-tests/src/main/java/io/casehub/connectors/playbook/ConnectorTestEngine.java` (register BankPlugin)
- Create: `connector-playbook-tests/src/main/resources/playbooks/voice-interaction-patterns/03-banking-assistant.playbook.yaml`
- Create: `connector-playbook-tests/src/main/resources/playbooks/voice-interaction-patterns/03-banking-assistant.simulation.yaml`
- Create: `connector-playbook-tests/src/test/resources/voice-interaction-patterns/03-banking-assistant.test.yaml`

**Interfaces:**
- Consumes: `BankPlatform`, `AccountInformation`, `RefBankPlatform`, `InMemoryBankBackend` from bank-spi/bank-ref
- Produces: `BankPlugin` (registered as `"bank"`) with operations: `balance`, `transactions`, `accounts`

- [ ] **Step 1: Write BankPlugin**

```java
package io.casehub.connectors.playbook.plugins;

import java.util.List;
import java.util.Map;

import io.casehub.connectors.bank.spi.BankPlatform;
import io.casehub.platform.yaml.plugin.Definition;
import io.casehub.platform.yaml.plugin.Result;

public class BankPlugin {

    public static Definition definition(BankPlatform bankPlatform) {
        return Definition.of("bank")
            .execute((inputs) -> {
                String operation = primaryInput(inputs);
                String userId = (String) inputs.getOrDefault("userId", "user-1");
                return switch (operation) {
                    case "balance" -> {
                        String accountId = (String) inputs.get("_value");
                        var balance = bankPlatform.accountInformation(userId).balance(accountId);
                        yield Result.of(Map.of(
                            "amount", balance.amount(),
                            "currency", balance.currency(),
                            "accountId", accountId
                        ));
                    }
                    case "transactions" -> {
                        String accountId = (String) inputs.get("_value");
                        var page = bankPlatform.accountInformation(userId).transactions(accountId, null);
                        yield Result.of(Map.of(
                            "items", page.items().stream()
                                .map(tx -> Map.of(
                                    "description", tx.description(),
                                    "amount", tx.amount(),
                                    "date", tx.date().toString()
                                ))
                                .toList()
                        ));
                    }
                    case "accounts" -> {
                        var accounts = bankPlatform.accountInformation(userId).accounts();
                        yield Result.of(Map.of("accounts", accounts));
                    }
                    default -> Result.failed("Unknown bank operation: " + operation);
                };
            })
            .build();
    }

    private static String primaryInput(Map<String, Object> inputs) {
        return (String) inputs.getOrDefault("_subject", inputs.get("operation"));
    }
}
```

Verify the exact field names on `Balance`, `Transaction` by reading the bank-spi model classes during implementation.

- [ ] **Step 2: Register BankPlugin in ConnectorTestEngine**

Add to ConnectorTestEngine's execute override:
```java
serviceRegistry.register(BankPlatform.class, new RefBankPlatform(new InMemoryBankBackend()));
pluginRegistry.register(BankPlugin.definition(serviceRegistry.lookup(BankPlatform.class)));
```

- [ ] **Step 3: Write 03-banking-assistant.playbook.yaml**

```yaml
name: Banking voice assistant
description: Customer asks about balance via voice, agent queries bank API
steps:
  - voice transcribe: "What is my current account balance?"
    language: en
    capture: transcript

  - agent-session-open banker:
      system: >
        You are a banking assistant. Extract the intent and respond with JSON.
        For balance queries respond: {"action": "check_balance", "account": "acc-001"}

  - agent-query banker: "{{transcript.text}}"
    capture: intent

  - bank balance: "{{intent.account}}"
    userId: user-1
    capture: balance

  - agent-query banker: "The balance for account {{balance.accountId}} is {{balance.amount}} {{balance.currency}}. Summarise this for the customer."
    capture: response

  - agent-session-close: banker

  - voice synthesise: "{{response.text}}"
    capture: audio
```

- [ ] **Step 4: Write 03-banking-assistant.simulation.yaml**

```yaml
methods:
  voice.transcribe:
    strategy: sequential
    corpus:
      - output:
          text: "What is my current account balance?"
          language: en
          confidence: 0.96
  voice.synthesise:
    strategy: sequential
    corpus:
      - output:
          audioData: []
          audioFormat: wav
  bank.balance:
    strategy: key-lookup
    corpus:
      - key: "acc-001"
        output:
          amount: 1250.00
          currency: GBP
          accountId: acc-001
```

- [ ] **Step 5: Write 03-banking-assistant.test.yaml**

```yaml
name: Banking voice assistant tests
simulate:
  agent:
    - prompt: "balance"
      response: '{"action": "check_balance", "account": "acc-001"}'
    - prompt: "balance for account"
      response: "Your current account balance is one thousand two hundred and fifty pounds."
tests:
  - name: "voice banking flow end to end"
    steps:
      - run-playbook: playbooks/voice-interaction-patterns/03-banking-assistant.playbook.yaml
        capture: result
      - assertThat: result.succeeded
        equals: true
      - assertThat: result.stepCount
        equals: 7

  - name: "bank balance step was called"
    steps:
      - run-playbook: playbooks/voice-interaction-patterns/03-banking-assistant.playbook.yaml
        capture: result
      - verify-simulation:
          method: bank.balance
          called: 1
```

- [ ] **Step 6: Run tests and verify**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn -pl connector-playbook-tests test -f /Users/mdproctor/claude/casehub/connectors/pom.xml`

Expected: 3 test suites pass (01 + 02 + 03), including two tests in 03.

- [ ] **Step 7: Commit**

```bash
git add connector-playbook-tests/
git commit -m "feat(#188): add BankPlugin + banking voice assistant playbook — voice + LLM + bank API"
```

---

## Batch 3: Chat + Email — Full Multi-Platform Showcase

After this batch, all 5 playbooks pass, demonstrating bi-directional pipelines across voice, bank, chat, and email with zero Java test code.

### Task 4: ChatPlugin + WhatsApp banking alert playbook

**Files:**
- Create: `connector-playbook-tests/src/main/java/io/casehub/connectors/playbook/plugins/ChatPlugin.java`
- Modify: `connector-playbook-tests/src/main/java/io/casehub/connectors/playbook/ConnectorTestEngine.java` (register ChatPlugin)
- Create: `connector-playbook-tests/src/main/resources/playbooks/voice-interaction-patterns/04-whatsapp-banking-alert.playbook.yaml`
- Create: `connector-playbook-tests/src/main/resources/playbooks/voice-interaction-patterns/04-whatsapp-banking-alert.simulation.yaml`
- Create: `connector-playbook-tests/src/test/resources/voice-interaction-patterns/04-whatsapp-banking-alert.test.yaml`

**Interfaces:**
- Consumes: `ChatPlatform`, `RefChatPlatform`, `InMemoryChatBackend` from chat-spi/chat-ref
- Produces: `ChatPlugin` (registered as `"chat"`) with operations: `send`, `channels`, `history`

- [ ] **Step 1: Write ChatPlugin**

```java
package io.casehub.connectors.playbook.plugins;

import java.util.Map;

import io.casehub.connectors.chat.spi.ChatPlatform;
import io.casehub.platform.yaml.plugin.Definition;
import io.casehub.platform.yaml.plugin.Result;

public class ChatPlugin {

    public static Definition definition(ChatPlatform chatPlatform) {
        return Definition.of("chat")
            .execute((inputs) -> {
                String operation = primaryInput(inputs);
                return switch (operation) {
                    case "send" -> {
                        String channelId = (String) inputs.get("_value");
                        String message = (String) inputs.get("message");
                        chatPlatform.messaging().send(channelId, message);
                        yield Result.of(Map.of("sent", true, "channel", channelId));
                    }
                    case "channels" -> {
                        var channels = chatPlatform.discovery().channels(null);
                        yield Result.of(Map.of("channels", channels.items()));
                    }
                    case "history" -> {
                        String channelId = (String) inputs.get("_value");
                        var messages = chatPlatform.messageHistory().messages(channelId, null, null, null);
                        yield Result.of(Map.of("messages", messages.items()));
                    }
                    default -> Result.failed("Unknown chat operation: " + operation);
                };
            })
            .build();
    }

    private static String primaryInput(Map<String, Object> inputs) {
        return (String) inputs.getOrDefault("_subject", inputs.get("operation"));
    }
}
```

Verify exact ChatPlatform capability accessor names (`messaging()`, `discovery()`, `messageHistory()`) and method signatures by reading chat-spi during implementation.

- [ ] **Step 2: Register ChatPlugin in ConnectorTestEngine**

Add to ConnectorTestEngine's execute override:
```java
serviceRegistry.register(ChatPlatform.class, new RefChatPlatform(new InMemoryChatBackend()));
pluginRegistry.register(ChatPlugin.definition(serviceRegistry.lookup(ChatPlatform.class)));
```

- [ ] **Step 3: Write 04-whatsapp-banking-alert.playbook.yaml**

```yaml
name: WhatsApp banking alert
description: Customer asks balance via WhatsApp chat, gets formatted reply with recent transactions
steps:
  - chat send: "general"
    message: "Checking your balance now..."

  - bank balance: "acc-001"
    userId: user-1
    capture: balance

  - bank transactions: "acc-001"
    userId: user-1
    capture: recent

  - agent-session-open formatter:
      system: >
        You are a message formatter. Given a balance and transactions,
        format a clear WhatsApp message. Use currency symbols and bullet points.

  - agent-query formatter: "Balance: {{balance.amount}} {{balance.currency}}. Transactions: {{recent.items}}"
    capture: formatted

  - agent-session-close: formatter

  - chat send: "general"
    message: "{{formatted.text}}"
```

- [ ] **Step 4: Write 04-whatsapp-banking-alert.simulation.yaml**

```yaml
methods:
  bank.balance:
    strategy: sequential
    corpus:
      - output:
          amount: 1250.00
          currency: GBP
          accountId: acc-001
  bank.transactions:
    strategy: sequential
    corpus:
      - output:
          items:
            - description: "Coffee Shop"
              amount: 4.50
              date: "2026-11-08"
            - description: "Grocery Store"
              amount: 32.10
              date: "2026-11-07"
            - description: "Monthly Salary"
              amount: 3200.00
              date: "2026-11-01"
  chat.send:
    strategy: sequential
    corpus:
      - output:
          sent: true
          channel: general
      - output:
          sent: true
          channel: general
```

- [ ] **Step 5: Write 04-whatsapp-banking-alert.test.yaml**

```yaml
name: WhatsApp banking alert tests
simulate:
  agent:
    - prompt: "Balance"
      response: >
        💰 Balance: £1,250.00

        📋 Recent transactions:
        • Coffee Shop: £4.50
        • Grocery Store: £32.10
        • Monthly Salary: £3,200.00
tests:
  - name: "chat banking flow sends two messages"
    steps:
      - run-playbook: playbooks/voice-interaction-patterns/04-whatsapp-banking-alert.playbook.yaml
        capture: result
      - assertThat: result.succeeded
        equals: true
      - verify-simulation:
          method: chat.send
          called: 2
      - verify-simulation:
          method: bank.balance
          called: 1
```

- [ ] **Step 6: Run tests and verify**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn -pl connector-playbook-tests test -f /Users/mdproctor/claude/casehub/connectors/pom.xml`

Expected: 4 test suites pass (01–04).

- [ ] **Step 7: Commit**

```bash
git add connector-playbook-tests/
git commit -m "feat(#188): add ChatPlugin + WhatsApp banking alert — bi-directional chat + bank"
```

### Task 5: EmailPlugin + email-voice notification playbook

**Files:**
- Create: `connector-playbook-tests/src/main/java/io/casehub/connectors/playbook/plugins/EmailPlugin.java`
- Modify: `connector-playbook-tests/src/main/java/io/casehub/connectors/playbook/ConnectorTestEngine.java` (register EmailPlugin)
- Create: `connector-playbook-tests/src/main/resources/playbooks/voice-interaction-patterns/05-email-voice-notification.playbook.yaml`
- Create: `connector-playbook-tests/src/main/resources/playbooks/voice-interaction-patterns/05-email-voice-notification.simulation.yaml`
- Create: `connector-playbook-tests/src/test/resources/voice-interaction-patterns/05-email-voice-notification.test.yaml`

**Interfaces:**
- Consumes: `EmailPlatform`, `RefEmailPlatform`, `InMemoryEmailBackend` from email-spi/email-ref
- Produces: `EmailPlugin` (registered as `"email"`) with operations: `messages`, `read`, `search`

- [ ] **Step 1: Write EmailPlugin**

```java
package io.casehub.connectors.playbook.plugins;

import java.time.Instant;
import java.util.Map;

import io.casehub.connectors.email.spi.EmailPlatform;
import io.casehub.platform.yaml.plugin.Definition;
import io.casehub.platform.yaml.plugin.Result;

public class EmailPlugin {

    public static Definition definition(EmailPlatform emailPlatform) {
        return Definition.of("email")
            .execute((inputs) -> {
                String operation = primaryInput(inputs);
                return switch (operation) {
                    case "messages" -> {
                        String mailboxId = (String) inputs.get("_value");
                        var page = emailPlatform.listMessages(mailboxId, null, null, null);
                        yield Result.of(Map.of(
                            "items", page.items().stream()
                                .map(msg -> Map.of(
                                    "id", msg.id(),
                                    "subject", msg.subject(),
                                    "from", msg.from()
                                ))
                                .toList()
                        ));
                    }
                    case "read" -> {
                        String messageId = (String) inputs.get("_value");
                        String mailboxId = (String) inputs.getOrDefault("mailbox", "INBOX");
                        var message = emailPlatform.getMessage(mailboxId, messageId);
                        yield Result.of(Map.of(
                            "id", message.id(),
                            "subject", message.subject(),
                            "from", message.from(),
                            "body", message.body()
                        ));
                    }
                    case "search" -> {
                        String query = (String) inputs.get("_value");
                        var results = emailPlatform.search(query, null);
                        yield Result.of(Map.of("items", results.items()));
                    }
                    case "mailboxes" -> {
                        var mailboxes = emailPlatform.listMailboxes();
                        yield Result.of(Map.of("mailboxes", mailboxes));
                    }
                    default -> Result.failed("Unknown email operation: " + operation);
                };
            })
            .build();
    }

    private static String primaryInput(Map<String, Object> inputs) {
        return (String) inputs.getOrDefault("_subject", inputs.get("operation"));
    }
}
```

Verify `EmailSummary` and `EmailMessage` field names by reading email-spi during implementation.

- [ ] **Step 2: Register EmailPlugin in ConnectorTestEngine**

Add to ConnectorTestEngine's execute override:
```java
serviceRegistry.register(EmailPlatform.class, new RefEmailPlatform(new InMemoryEmailBackend()));
pluginRegistry.register(EmailPlugin.definition(serviceRegistry.lookup(EmailPlatform.class)));
```

- [ ] **Step 3: Write 05-email-voice-notification.playbook.yaml**

```yaml
name: Email and voice notification
description: Read latest email, summarise via agent, deliver as voice notification
steps:
  - email messages: "INBOX"
    capture: inbox

  - agent-session-open summariser:
      system: >
        You are an email summariser. Given an email subject and sender,
        produce a one-sentence spoken summary suitable for a voice notification.

  - agent-query summariser: "Summarise: From {{inbox.items[0].from}}, Subject: {{inbox.items[0].subject}}"
    capture: summary

  - agent-session-close: summariser

  - voice synthesise: "{{summary.text}}"
    capture: voiceAlert

  - chat send: "notifications"
    message: "📧 Email summary: {{summary.text}}"
```

- [ ] **Step 4: Write 05-email-voice-notification.simulation.yaml**

```yaml
methods:
  email.messages:
    strategy: sequential
    corpus:
      - output:
          items:
            - id: "msg-001"
              subject: "Large transaction alert"
              from: "bank-alerts@example.com"
            - id: "msg-002"
              subject: "Meeting tomorrow at 10am"
              from: "colleague@example.com"
  voice.synthesise:
    strategy: sequential
    corpus:
      - output:
          audioData: []
          audioFormat: wav
  chat.send:
    strategy: sequential
    corpus:
      - output:
          sent: true
          channel: notifications
```

- [ ] **Step 5: Write 05-email-voice-notification.test.yaml**

```yaml
name: Email voice notification tests
simulate:
  agent:
    - prompt: "Large transaction"
      response: "You received a large transaction alert from your bank."
tests:
  - name: "email to voice notification pipeline"
    steps:
      - run-playbook: playbooks/voice-interaction-patterns/05-email-voice-notification.playbook.yaml
        capture: result
      - assertThat: result.succeeded
        equals: true
      - verify-simulation:
          method: email.messages
          called: 1
      - verify-simulation:
          method: voice.synthesise
          called: 1
      - verify-simulation:
          method: chat.send
          called: 1

  - name: "cross-platform: email + voice + chat all invoked"
    steps:
      - run-playbook: playbooks/voice-interaction-patterns/05-email-voice-notification.playbook.yaml
        capture: result
      - assertThat: result.stepCount
        equals: 6
```

- [ ] **Step 6: Run all tests and verify**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn -pl connector-playbook-tests test -f /Users/mdproctor/claude/casehub/connectors/pom.xml`

Expected: All 5 test suites pass (01–05), demonstrating voice, agent, bank, chat, and email integration with zero Java test code.

- [ ] **Step 7: Commit**

```bash
git add connector-playbook-tests/
git commit -m "feat(#188): add EmailPlugin + email-voice-notification — full multi-platform showcase"
```

---

## Batch 4: Full build verification

### Task 6: Full build + docs

**Files:**
- Modify: `docs/guides/consumer-guide.md` (add playbook tests section)

- [ ] **Step 1: Run full project build**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn clean install -f /Users/mdproctor/claude/casehub/connectors/pom.xml`

Expected: All modules build and all tests pass, including the 5 new playbook test suites.

- [ ] **Step 2: Update consumer guide**

Add a section to `docs/guides/consumer-guide.md` documenting the playbook test module — what it provides, how to write new scenarios, the domain plugin syntax.

- [ ] **Step 3: Commit**

```bash
git add docs/ connector-playbook-tests/
git commit -m "docs(#188): add playbook tests documentation to consumer guide"
```

## References

- [specs/voice-playbook-tests/2026-11-09-voice-playbook-tests-design.md] — design spec this plan implements
- [voice-spi/src/main/java/io/casehub/connectors/voice/spi/VoicePlatform.java] — VoicePlatform SPI
- [voice-ref/src/main/java/io/casehub/connectors/voice/ref/ReferenceVoicePlatform.java] — reference impl
- [bank-spi/src/main/java/io/casehub/connectors/bank/spi/BankPlatform.java] — BankPlatform SPI
- [chat-spi/src/main/java/io/casehub/connectors/chat/spi/ChatPlatform.java] — ChatPlatform SPI
- [email-spi/src/main/java/io/casehub/connectors/email/spi/EmailPlatform.java] — EmailPlatform SPI
- Platform YamlTestEngine — yaml-test module
- Platform Walker combined-key — yaml-step-runtime Walker.java lines 280-305
- Platform SimulationRuntime — simulation-core
- [GitHub #188] — real-time inbound channels
