# HANDOFF — casehub-connectors

## Last Session

Completed zero-Java playbook test migration and built a kitchen-sink voice processing pipeline. Platform fixes landed for Walker inputKey (#777 Gap 1, committed `d831b840`) and AptPluginSource required fields (committed `e88262fe`). Three-line `ConnectorPlaybookSuiteTest` replaces 130-line manual Java test class.

**Delivered:**
- Zero-Java Suite: `@Suite @IncludeEngines("casehub-yaml") @SelectDirectories`
- `AudioPlugin` — join, trim, splice, mix via `javax.sound.sampled`
- `AudioDenoisePlugin` — SpeechDenoiser wrapper
- `SpeakerPlugin` — register/identify via SpeakerEmbeddingExtractor + SpeakerRegistry
- `ChatSendPlugin` attachment support — `@Optional byte[]` field, type-safe expression flow
- Ref implementations: `RefSpeechDenoiser`, `RefSpeakerEmbeddingExtractor`, `RefSpeakerRegistry`
- Per-module `yaml-services.yaml` in voice-ref, bank-ref, chat-ref, email-ref
- APT processor on all 4 SPI modules (voice-pipeline-steps, bank-spi, chat-spi, email-spi)
- ReferenceVoicePlatform voice tagging (`[voice:name]` in WAV data)
- All 6 playbooks updated: unquoted YAML, relative paths, voice messages with attachments, transcript follow-ups
- Blog revised: "YAML All the Way Down" with voice pipeline SVG

**Platform commits this session:**
- `d831b840` — Walker streaming inputKey fallback (#777)
- `e88262fe` — AptPluginSource reads required fields from JSON Schema (#777)

## Immediate Next Step

**Voice config manifest (#203)** — provider routing and voice registry following the agent-config pattern. Priority-based accumulation from classpath → `/etc/casehub/` → `~/.casehub/` → project dir. Maps voice names to provider-specific IDs (Kokoro integer IDs, Whisper model paths, etc.). Same credential resolution as LLM manifest.

## Cross-Module

- **Enabled** — platform#777 (Walker + AptPluginSource gaps) landed; zero-Java Suite works end-to-end
- **Blocked by** — none

## References

- Branch: `issue-188-real-time-inbound-channels` at `288845a0`
- Blog: `blog/2026-11-09-mdp01-yaml-all-the-way-down.md`
- Platform custom step guide: `platform/docs/guides/custom-step-authoring.md`
- Issues filed: connectors#201 (ChatSendPlugin attachments), connectors#202 (voice manifest — superseded), connectors#203 (voice config manifest)
- Platform issues: #776 (byte[] fix — closed), #777 (Walker + required fields — landed)
