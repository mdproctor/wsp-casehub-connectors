# Voice Playbook Integration Tests — Design Spec

## Goal

End-to-end YAML playbook integration tests demonstrating voice + multi-platform scenarios (banking, WhatsApp, email) with zero Java test code. Pure YAML playbooks, simulation files, and test files — the only Java is reusable infrastructure (test engine, domain plugins).

Dual-mode: simulation (default, offline, deterministic) and real LLM (profile-gated, manifest-based).

## Architecture

### ConnectorTestEngine

Extends `YamlTestEngine`. Overrides service registration to pre-register all connector reference platforms:

- `ReferenceVoicePlatform` → voice domain
- `ReferenceBankPlatform` → bank domain
- `ReferenceChatPlatform` → chat domain (WhatsApp, Slack simulation)
- `ReferenceEmailPlatform` → email domain
- `ReferenceCalendarPlatform` → calendar domain (available for future scenarios)

Each platform is wrapped by `SimulationRuntime` when a `.simulation.yaml` is paired, so simulation strategies (sequential, key-lookup, state-machine) apply transparently.

Discovers `.test.yaml` files from the test classpath. JUnit Platform picks up `ConnectorTestEngine` via `META-INF/services/org.junit.platform.engine.TestEngine`.

### Domain Plugins

One `@Plugin` per connector domain. Walker's combined-key parsing routes operations automatically.

**VoicePlugin** — registered as `voice`:
```yaml
- voice transcribe: audio_bytes    # → speechToText().transcribe(audio)
  language: en
  capture: transcript

- voice synthesise: "Hello world"  # → textToSpeech().synthesise(text)
  voice: en-GB
  capture: audio
```

Dispatcher maps `primaryInput`:
- `transcribe` → `voicePlatform.speechToText().transcribe(value, options)`
- `synthesise` → `voicePlatform.textToSpeech().synthesise(value, options)`

**BankPlugin** — registered as `bank`:
```yaml
- bank balance: "acc-123"         # → accountInformation(userId).balance(accountId)
  userId: user1
  capture: balance

- bank transactions: "acc-123"    # → accountInformation(userId).transactions(accountId, pageRequest)
  userId: user1
  capture: txns
```

**ChatPlugin** — registered as `chat`:
```yaml
- chat send: "#general"           # → send(channelId, message)
  message: "Hello from the pipeline"
  capture: sent

- chat channels:                  # → channels(pageRequest)
  capture: channels
```

**EmailPlugin** — registered as `email`:
```yaml
- email send:                     # → connector.send(destination, message)
  to: user@example.com
  subject: "Balance alert"
  body: "Your balance is {{balance.amount}}"
  capture: sent

- email messages: "INBOX"         # → emailPlatform.messages(mailbox, pageRequest)
  capture: inbox
```

### Playbook Scenarios

Progressive examples in `src/main/resources/playbooks/voice-interaction-patterns/`:

**01-voice-echo.playbook.yaml** — pure voice round-trip:
```yaml
name: Voice echo
description: Transcribe speech then synthesise the transcript back
steps:
  - voice transcribe: "{{input.audio}}"
    capture: transcript
  - voice synthesise: "{{transcript.text}}"
    capture: audio
```

**02-voice-agent.playbook.yaml** — voice + LLM:
```yaml
name: Voice agent
description: Voice input → agent classification → voice output
steps:
  - voice transcribe: "{{input.audio}}"
    capture: transcript
  - agent-session-open classifier:
      system: You are a document classifier. Respond with one word.
  - agent-query classifier: "{{transcript.text}}"
    capture: classification
  - agent-session-close: classifier
  - voice synthesise: "Classified as {{classification.text}}"
    capture: audio
```

**03-banking-assistant.playbook.yaml** — voice + LLM + bank API:
```yaml
name: Banking voice assistant
description: Customer asks about balance via voice, agent queries bank
steps:
  - voice transcribe: "{{input.audio}}"
    capture: transcript
  - agent-session-open banker:
      system: >
        You are a banking assistant. When asked about balance,
        respond with JSON: {"action": "check_balance", "account": "<id>"}.
  - agent-query banker: "{{transcript.text}}"
    capture: intent
  - bank balance: "{{intent.account}}"
    userId: "{{input.userId}}"
    capture: balance
  - agent-query banker: "Tell the customer their balance is £{{balance.amount}}"
    capture: response
  - agent-session-close: banker
  - voice synthesise: "{{response.text}}"
    capture: audio
```

**04-whatsapp-banking-alert.playbook.yaml** — inbound chat + bank + outbound chat:
```yaml
name: WhatsApp banking alert
description: Customer asks balance via WhatsApp, gets formatted reply
steps:
  - chat send: "{{input.channel}}"
    message: "Checking your balance..."
  - bank balance: "{{input.accountId}}"
    userId: "{{input.userId}}"
    capture: balance
  - bank transactions: "{{input.accountId}}"
    userId: "{{input.userId}}"
    limit: 3
    capture: recent
  - chat send: "{{input.channel}}"
    message: >
      💰 Balance: £{{balance.amount}}
      📋 Recent:
      {{#recent.items}}
      - {{description}}: £{{amount}}
      {{/recent.items}}
```

**05-email-voice-notification.playbook.yaml** — bank event → email + voice:
```yaml
name: Email and voice notification
description: Large transaction triggers email alert and voice call summary
steps:
  - bank transactions: "{{input.accountId}}"
    userId: "{{input.userId}}"
    limit: 1
    capture: latest
  - email send:
    to: "{{input.userEmail}}"
    subject: "Large transaction alert"
    body: "Transaction of £{{latest.items[0].amount}} on {{latest.items[0].date}}"
    format: html
  - voice synthesise: "Alert: large transaction of £{{latest.items[0].amount}} detected"
    capture: voiceAlert
```

### Simulation Files

Each playbook paired with a `.simulation.yaml`:

**03-banking-assistant.simulation.yaml**:
```yaml
methods:
  voice.transcribe:
    strategy: sequential
    corpus:
      - output:
          text: "What's my current account balance?"
          confidence: 0.95
  voice.synthesise:
    strategy: sequential
    corpus:
      - output:
          audio: [silent-wav-bytes]
          duration: 2.5
  bank.balance:
    strategy: key-lookup
    corpus:
      - key: "acc-123"
        output:
          amount: 1250.00
          currency: GBP
          accountId: acc-123
  bank.transactions:
    strategy: sequential
    corpus:
      - output:
          items:
            - description: "Coffee Shop"
              amount: 4.50
              date: "2026-11-08"
```

### Test Files

**03-banking-assistant.test.yaml**:
```yaml
name: Banking voice assistant tests
simulate:
  agent:
    - prompt: "balance"
      response: '{"action": "check_balance", "account": "acc-123"}'
    - prompt: "Tell the customer"
      response: "Your current account balance is twelve hundred and fifty pounds."
tests:
  - name: "voice banking flow succeeds end to end"
    vars:
      input.audio: [test-audio-bytes]
      input.userId: user1
    steps:
      - run-playbook: playbooks/voice-interaction-patterns/03-banking-assistant.playbook.yaml
        capture: result
      - assertThat: result.succeeded
        equals: true
      - assertThat: result.stepCount
        equals: 7
      - verify-simulation:
          method: voice.transcribe
          called: 1
      - verify-simulation:
          method: bank.balance
          called: 1
      - verify-simulation:
          method: voice.synthesise
          called: 1
```

### Module Structure

```
connector-playbook-tests/
  pom.xml
  src/main/java/io/casehub/connectors/playbook/
    ConnectorTestEngine.java
    plugins/
      VoicePlugin.java
      BankPlugin.java
      ChatPlugin.java
      EmailPlugin.java
  src/main/resources/
    META-INF/services/org.junit.platform.engine.TestEngine
    playbooks/voice-interaction-patterns/
      01-voice-echo.playbook.yaml
      01-voice-echo.simulation.yaml
      02-voice-agent.playbook.yaml
      02-voice-agent.simulation.yaml
      03-banking-assistant.playbook.yaml
      03-banking-assistant.simulation.yaml
      04-whatsapp-banking-alert.playbook.yaml
      04-whatsapp-banking-alert.simulation.yaml
      05-email-voice-notification.playbook.yaml
      05-email-voice-notification.simulation.yaml
  src/test/resources/
    01-voice-echo.test.yaml
    02-voice-agent.test.yaml
    03-banking-assistant.test.yaml
    04-whatsapp-banking-alert.test.yaml
    05-email-voice-notification.test.yaml
```

### Dependencies (test scope where appropriate)

```xml
<!-- Platform infrastructure -->
<dependency>casehub-platform-yaml-test</dependency>
<dependency>casehub-platform-yaml-step-runtime</dependency>
<dependency>casehub-platform-simulation-core</dependency>
<dependency>casehub-platform-playbook-compiler</dependency>
<dependency>casehub-platform-yaml-plugin-api</dependency>

<!-- Connector reference implementations -->
<dependency>voice-spi</dependency>
<dependency>voice-ref</dependency>
<dependency>bank-spi</dependency>
<dependency>bank-ref</dependency>
<dependency>chat-spi</dependency>
<dependency>chat-ref</dependency>
<dependency>email-spi</dependency>
<dependency>email-ref</dependency>
```

### Testing Modes

**Simulation (default):** `mvn test` — all platforms simulated, LLM via corpus in simulate: block. Deterministic, offline, fast.

**Real LLM (-Preal-llm):** `mvn test -Preal-llm` — manifest-based LLM backend, platform APIs still simulated. Validates playbook logic with actual LLM reasoning.

## Implementation Order

1. **ConnectorTestEngine** — extend YamlTestEngine, register reference platforms, wire SimulationRuntime
2. **Domain plugins** — VoicePlugin, BankPlugin, ChatPlugin, EmailPlugin with combined-key dispatch
3. **Playbook 01** — voice-echo (simplest, validates engine + voice plugin)
4. **Playbook 02** — voice-agent (adds LLM simulation)
5. **Playbook 03** — banking-assistant (adds bank plugin + multi-step)
6. **Playbook 04** — whatsapp-banking-alert (adds chat plugin + bi-directional)
7. **Playbook 05** — email-voice-notification (adds email plugin + cross-platform)
8. **Real LLM profile** — manifest integration, profile-gated test config

## References

- Platform YamlTestEngine: `yaml-test/src/main/java/.../YamlTestEngine.java`
- Platform Walker combined-key: `yaml-step-runtime/.../Walker.java` lines 280-305
- Platform SimulationRuntime: `simulation-core/.../SimulationRuntime.java`
- Platform playbook examples: `playbooks/src/main/playbooks/agent-interaction-patterns/`
- Connectors voice-spi: `voice-spi/` (on issue-188 branch)
- Connectors voice-ref: `voice-ref/` (on issue-188 branch)
- Connectors bank-ref: `bank-ref/`
- Connectors chat-ref: `chat-ref/`
- Connectors email-ref: `email-ref/`
