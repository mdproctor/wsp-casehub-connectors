---
layout: post
title: "Voice Needs a Manifest"
date: 2026-11-09
entry_type: article
subtype: diary
projects: [casehubio/connectors]
tags: [voice, config, routing, registry, spi, agent-config, yaml, architecture]
series: issue-188-real-time-inbound-channels
---

# Voice Needs a Manifest

A playbook step that says `voice: af_heart` should not need to know that Heart is a Kokoro voice, that Kokoro needs a model directory at `/models/kokoro-multi-lang-v1_0`, or that the voice-id Kokoro understands is `0`. The step names a voice. Something else resolves it.

The agent side of the platform already solved this problem. `agent-config.yaml` declares LLM providers and models in a priority chain — classpath seed, system config, user config, project override. `ModelRegistry` catalogs descriptors. `RoutingAgentProvider` resolves a model reference (by name, alias, tier, or criteria query) to the right backend, transparently. The caller says `model: fast` and the router picks Claude Haiku on the configured Anthropic backend.

Voice needs the same thing.

## The problem with hardcoded vendor knowledge

The voice pipeline steps built earlier in this branch take a `voice` string and pass it straight through to the underlying `VoicePlatform`:

```java
var options = new SynthesisOptions(voice, language, "wav", false);
return voicePlatform.textToSpeech().synthesise(text, options);
```

This works when there's exactly one TTS vendor. The moment you have two — say Kokoro for local inference and OpenAI for cloud fallback — the caller needs to know which vendor owns which voice name. The playbook step becomes vendor-aware, which violates the entire point of having a platform abstraction.

The agent config pattern already showed that the fix is a manifest: declare the voices, declare the vendors, let a registry and a router handle the mapping.

## The pattern mirror

I built the voice config as a direct structural mirror of the agent config. Every type has an exact analogue:

| Agent Config | Voice Config |
|---|---|
| `ModelDescriptor` | `VoiceDescriptor` |
| `ModelRegistry` | `VoiceRegistry` |
| `ModelQuery` | `VoiceQuery` |
| `ManifestLoader` | `VoiceManifestLoader` |
| `RoutingAgentProvider` | `RoutingVoicePlatformService` |

The voice manifest loader reads `voice-config.yaml` from the same six-level priority chain — classpath seed catalog at priority 0, `/etc/casehub/` at 10, `~/.casehub/` at 20, profile variants at 25, project directory at 30/35. Higher priority overrides lower, keyed by voice name for voices, vendor for providers, alias name for aliases. The merge is a `LinkedHashMap` sorted by priority — identical to what the agent loader does.

![Voice config routing flow](images/voice-config-routing.svg)

## What a voice-config.yaml looks like

```yaml
providers:
  - vendor: kokoro
    model-dir: ${KOKORO_MODEL_DIR:/models/kokoro-multi-lang-v1_0}
  - vendor: whisper
    model-dir: ${WHISPER_MODEL_DIR:/models/whisper-large-v3}

voices:
  af_heart:
    vendor: kokoro
    voice-id: "0"
    name: Heart
    gender: female
    accent: american
    quality: 4.5
    languages: [en-US]
    capability: tts

aliases:
  alice:
    voice: af_heart
  high-quality:
    min-quality: 4.0
    prefer-vendor: kokoro
    capability: tts

defaults:
  tts-vendor: kokoro
  stt-vendor: whisper
```

Three resolution modes. By name: `voice: af_heart` looks up the descriptor directly. By alias: `voice: alice` follows the alias to `af_heart`. By criteria: `voice: high-quality` queries the registry for all TTS voices with quality ≥ 4.0, preferring the Kokoro vendor, and picks the best match.

The seed catalog that ships with the SPI maps the Kokoro voice names to the `ref` vendor — the in-memory reference platform. A deployment that wants real Kokoro inference drops a `voice-config.yaml` in the project directory that maps the same voice names to the `kokoro` vendor. The playbook YAML doesn't change. The voice name is an abstraction that the manifest resolves.

## What this changes for the plugin

The TTS plugin used to take a `VoicePlatform` directly. Now it takes a `RoutingVoicePlatformService`:

```java
@Execute
public SynthesisResult apply(String text, RoutingVoicePlatformService routing) {
    if (voice != null) {
        var resolved = routing.resolve(voice);
        var options = new SynthesisOptions(
            resolved.descriptor().voiceId(), language, "wav", false);
        return resolved.platform().textToSpeech().synthesise(text, options);
    }
    return routing.defaultTts().textToSpeech().synthesise(text, ...);
}
```

`resolve("alice")` returns a `ResolvedVoice` — the correct `VoicePlatform` engine for that voice's vendor, plus the `VoiceDescriptor` with the vendor-specific voice-id. The plugin doesn't need to know that Alice is a Kokoro voice. It doesn't need to know what Kokoro is.

The same applies to STT. `routing.defaultStt()` returns the platform for the configured default STT vendor. A playbook that transcribes audio doesn't pick an engine — the manifest picks it.

## Why not extend the agent manifest?

Voice config lives in connector infrastructure, not agent infrastructure. The agent manifest (`agent-config.yaml`) belongs to `agent-config-core` in the platform. Adding voice sections to it would couple `voice-spi` to `agent-config-core` — a dependency that doesn't exist and shouldn't. The priority-chain pattern is simple enough (six `addFileIfExists` calls and a `LinkedHashMap` merge) that duplicating it costs less than the coupling.

A shared utility in `platform-api` was the other option. But a 20-line pattern doesn't justify a new platform abstraction. If a third config type needs the same chain, that's the time to extract.

## The alias cycle guard

One difference from the agent pattern: voice aliases can reference other aliases, which means cycles are possible. `alice → bob → alice` would stack-overflow without protection. The resolver tracks a visited set and throws on a cycle — something the agent router doesn't need because model chains are flat lists, not recursive references.

The code review caught this. A cycle in a config file is a deployment error — the right response is a clear error message at startup, not a silent stack overflow at the first TTS call.
