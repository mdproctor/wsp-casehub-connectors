# HANDOFF — casehub-connectors

## Last Session

Completed voice config manifest (#203) — priority-chain YAML loading, voice registry, and routing voice platform service. All types in voice-spi, mirroring the agent-config pattern. Code review done with 7 findings fixed (cycle guard, null checks, test quality, CDI wiring). Blog written: "Voice Needs a Manifest."

**Delivered:**
- `VoiceDescriptor`, `VoiceCapability`, `VoiceQuery` — model types with builder
- `VoiceRegistry`, `MutableVoiceRegistry`, `InMemoryVoiceRegistry` — catalog interfaces and implementation
- `VoiceManifest`, `VoiceProviderDeclaration`, `VoiceAliasDeclaration`, `VoiceManifestDefaults` — YAML config types
- `VoiceManifestLoader` — 6-level priority-chain loading (classpath → /etc → ~/.casehub → profile → project)
- `VoiceManifestProcessor`, `VoiceManifestResult` — manifest-to-registry processing
- `RoutingVoicePlatformService`, `ResolvedVoice` — resolve by name/alias/criteria with cycle guard
- `VoiceConfigProducer` — CDI wiring with profile and project-dir support
- Seed catalog (`voices/seed-catalog.yaml`) — maps af_heart/bm_daniel to ref platform
- `TtsSynthesisePlugin` and `SttTranscribePlugin` migrated to `RoutingVoicePlatformService`
- All integration tests updated and passing (90 tests across 4 modules)
- Blog: "Voice Needs a Manifest" with SVG routing diagram

**Prior session (same branch):**
- Zero-Java Suite, AudioPlugin, SpeakerPlugin, AudioDenoisePlugin
- Per-module yaml-services.yaml, APT processor on 4 SPI modules
- All 6 playbooks updated, blog "YAML All the Way Down"

## Immediate Next Step

**Close GitHub issue #203** — work is complete, merged to branch. Then either continue with next issue on the branch or `work end` if #188 scope is complete.

## Cross-Module

- **Enabled** — platform#777 (Walker + AptPluginSource gaps) landed; zero-Java Suite works end-to-end
- **Blocked by** — none

## Deferred

- SherpaVoicePlatform adapter (agency cross-repo) — requires IntelliJ workspace spanning connectors+agency

## References

- Branch: `issue-188-real-time-inbound-channels` at `6057c0c`
- Spec: `specs/issue-188-real-time-inbound-channels/2026-11-09-voice-config-manifest-design.md`
- Plan: `plans/2026-11-09-voice-config-manifest.md`
- Blog: `blog/2026-11-09-mdp02-voice-needs-a-manifest.md`
- Issues: connectors#203 (voice config manifest — done)
