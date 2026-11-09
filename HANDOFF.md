# HANDOFF — casehub-connectors

## Last Session

Built end-to-end YAML playbook integration tests (#188) — voice + multi-platform scenarios with zero Java test code.

**Delivered:**
- `playbooks/` module — 6 YAML playbooks, 6 YAML test files, 1 Suite class
- 5 `@Plugin` records in SPI modules: `SttTranscribePlugin`, `TtsSynthesisePlugin` (voice-pipeline-steps), `BankBalancePlugin` (bank-spi), `ChatSendPlugin` (chat-spi), `EmailMessagesPlugin` (email-spi)
- `ReferenceVoicePlatform` text round-trip encoding (TTS encodes text as bytes, STT decodes back)
- No-arg constructors on `RefBankPlatform`, `RefChatPlatform`, `RefEmailPlatform`
- Content assertions via `result.captures.*` — precise, traceable, no flat-merged magic keys

**Platform issues filed and fixed during session:**
- #769: ServiceRegistry forwarding to per-test runners — **landed**
- #770: Captures in PlaybookRunResult + deprecate flat merge — **landed**
- #771: Streaming primaryInputKey fallback — **landed**
- #772: Use `_value` convention for streaming input — **landed**
- #773: Auto-unwrap record return types to Maps (kebab-case) — **landed**

**Connectors issues:**
- #197: ReferenceVoicePlatform text round-trip — **fixed this session**

## Immediate Next Step

**Complete the zero-Java test runner migration.** Platform custom-step-authoring.md documents the full approach:

1. **APT processor on SPI modules** — partially done (build sections added to bank-spi, chat-spi, email-spi, voice-pipeline-steps POMs but need verification — IntelliJ kept switching branches causing edits to land on wrong branch)
2. **`ConnectorPlaybookSuite`** replaces `ConnectorPlaybookTest` — 3-line `@Suite @IncludeEngines("casehub-yaml") @SelectDirectories("src/test/playbooks")` class. Already written but not committed due to branch switching.
3. **`yaml-services.yaml`** in `src/main/resources/META-INF/` (not test — serves demo/dev too) maps SPI → ref implementation
4. **Service profiles** — `yaml-services-integration.yaml` for different seeding configs
5. **`Seedable` interface** on ref implementations — enables corpus-driven seeding for `reference` strategy mode

Key insight from session: `yaml-services.yaml` is not test-only. It configures the default service implementations for demo, dev, AND test environments. Ref implementation dependencies should be `runtime` scope, not `test`.

**CAUTION:** IntelliJ's Maven reimport switches the active branch in the connectors project. Multiple times during this session, edits landed on `main` or `mcp/typed-returns` instead of `issue-188`. Verify the branch before every commit.

## Playbook Scenarios

| # | Playbook | Domains | Pattern |
|---|----------|---------|---------|
| 01 | voice-echo | Voice | TTS→STT round-trip — text survives |
| 02 | voice-agent | Voice + LLM | TTS→STT→agent classify→TTS |
| 03 | banking-assistant | Voice + Bank | TTS→STT→bank balance→TTS |
| 04 | whatsapp-banking-alert | Chat + Bank | chat send→bank balance→chat reply |
| 05 | email-voice-notification | Email + Voice + Chat | inbox→TTS→chat notify |
| 06 | joint-account-confirmation | Voice | Two speakers confirm, both round-trip verified |

## Architecture

```
playbooks/
  src/main/playbooks/voice-interaction-patterns/   # YAML playbooks
  src/main/resources/META-INF/yaml-services.yaml   # SPI → ref impl mapping
  src/test/playbooks/voice-interaction-patterns/   # YAML test files
  src/test/java/.../ConnectorPlaybookSuite.java    # 3-line Suite class
```

**Plugin discovery:** `AptPluginSource` scans `META-INF/yaml-plugins/*.json` from classpath → auto-registers all `@Plugin` records.

**Service resolution:** `yaml-services.yaml` maps SPI interfaces to implementations → instantiated via no-arg constructor → registered in `ServiceRegistry` → forwarded to per-test runners (#769).

**Captures:** `result.captures.roundTrip.text` not `result.output.text` — assertions reference named captures from playbook YAML, no flat-merged magic keys (#770).

## References

- Branch: `issue-188-real-time-inbound-channels` at `2a5a184`
- Platform custom step authoring guide: `platform/docs/guides/custom-step-authoring.md`
- Spec: `specs/voice-playbook-tests/2026-11-09-voice-playbook-tests-design.md`
- Plan: `plans/2026-11-09-voice-playbook-tests.md`
- Platform issues: #769, #770, #771, #772, #773 (all landed)
- Connectors issues: #197 (fixed)
