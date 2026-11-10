# HANDOFF — casehub-connectors

## Last Session

Landed voice infrastructure on main — rebased issue-188 branch (52 commits), fast-forward merged. Voice config manifest (#203), zero-Java playbook suite, routing service, integration tests all on main.

**This session:**
- Rebased issue-188-real-time-inbound-channels onto main (resolved pom.xml module conflicts)
- Fast-forward merged to main — 143 files, 5967 additions
- Decided to migrate speech implementations from agency to connectors (connectors#204)
- Closed agency#445 (adapter approach superseded), filed agency#446 (agency cleanup)
- Agency's SpeechToTextService/TextToSpeechService will be deleted — redundant with VoicePlatform

**Key decision:** Voice model provisioning follows agency's auto-download pattern (Provisioner), not Maven artifacts. Kokoro is one shared 88-310MB ONNX model + small voice style vectors, not per-voice files.

## Immediate Next Step

**connectors#204 — Migrate speech implementations from agency to connectors.** Move Provisioner, native lib modules, KokoroTextToSpeech, VitsTextToSpeech, SherpaOnnxSpeechToText, WhisperSpeechToText from agency speech-sherpa into connectors as VoicePlatform modules (voice-kokoro, voice-vits, stt-whisper, etc.).

Also: move seed-catalog.yaml from voice-spi to voice-ref or playbooks — it hardcodes ref-specific config in the SPI module. Switch VoiceManifestLoader to scan META-INF/voice-catalog.yaml (multi-resource, like yaml-services.yaml).

## Cross-Module

- **Enabled** — platform#769-773, #777 all landed
- **Blocked by** — none
- agency#446 blocked by connectors#204

## References

- Issues: connectors#204 (speech migration), agency#446 (agency cleanup)
- Blog: blog/2026-11-09-mdp01-yaml-all-the-way-down.md
- Blog: blog/2026-11-09-mdp02-voice-needs-a-manifest.md
- Design specs: specs/issue-188-real-time-inbound-channels/
