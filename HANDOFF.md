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

**Real TTS with Kokoro ONNX models.** The manifest and routing are in place. Next is a production VoicePlatform implementation that loads ONNX models and produces real audio.

**Voice models:**
- `af_heart` — female (Alice in the joint account playbook)
- `bm_daniel` — male (Bob in the joint account playbook)
- Kokoro v1.0 ONNX — ~80MB per model

**Model management — Maven artifact in GitHub Packages:**
Publish models as Maven artifacts to `maven.pkg.github.com/casehubio/*`. CI already has `GITHUB_TOKEN` for this repo. Models resolve like any dependency — cached in `.m2`, fast on warm builds, no external URL dependency.

```xml
<dependency>
  <groupId>io.casehub.models</groupId>
  <artifactId>kokoro-v1.0</artifactId>
  <version>1.0</version>
  <type>onnx</type>
  <scope>test</scope>
</dependency>
```

One-time upload to Packages, same retrieval path as all other casehubio artifacts.

**Implementation:**
- `KokoroVoicePlatform` — loads ONNX model via `onnxruntime-java`, implements `VoicePlatform`
- Voice manifest entry maps `af_heart`/`bm_daniel` to Kokoro provider
- Playbooks already use `voice: af_heart` / `voice: bm_daniel` — no YAML changes needed
- Round-trip tests become real integration tests: TTS produces actual WAV, STT (via Whisper/Sherpa) transcribes back

**Key insight from prior session:** Reference implementations are not test stubs — they serve demo, dev, and onboarding environments. `yaml-services.yaml` is the default service config across all environments. Service profiles (`-Dcasehub.yaml.services.profile=integration`) swap implementations without changing playbooks.

## Cross-Module

- **Enabled** — platform#769-773, #777 all landed; zero-Java Suite works end-to-end
- **Blocked by** — none

## Deferred

- SherpaVoicePlatform adapter (agency cross-repo) — requires IntelliJ workspace spanning connectors+agency

## References

- Branch: `issue-188-real-time-inbound-channels`
- Platform custom step authoring: `platform/docs/guides/custom-step-authoring.md`
- Blog: `blog/2026-11-09-mdp01-yaml-all-the-way-down.md`
- Blog: `blog/2026-11-09-mdp02-voice-needs-a-manifest.md`
- Issues: connectors#197 (round-trip — done), #203 (voice manifest — done)
- Platform issues: #769, #770, #771, #772, #773, #777 (all landed)
