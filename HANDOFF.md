# HANDOFF — casehub-connectors

## Last Session

Landed #264 — added `@GeneratePlugin` annotations to voice and messaging methods. Landed as c3ca2859 on main, published.

- Voice: `@GeneratePlugin(spi = VoicePlatform.class)` on `speak`, `transcribe`, `listVoices` — the `RoutingVoicePlatformService` routing pattern is invisible to AST inference
- Messaging: `@GeneratePlugin(exclude = true)` on `sendEmail`, `sendSlack`, `sendTeams` — legacy `ConnectorService.send()` dispatch, not worth teaching the scanner

Blog entry written: `blog/2026-11-18-mdp02-the-pattern-the-generator-couldnt-follow.md`

## Filed Issues

- #295 — `ConnectorTravelApi` has unused `GeneratePlugin` import (XS/Low)
- #296 — Audit all `@McpDomain` APIs for `@GeneratePlugin` coverage (S/Med)
- #297 — Issue #264 says 5 messaging methods but code has 3 (XS/Low)

## Immediate Next Step

Queue has 14 remaining items. First unchecked is #288 (fix-playbook-integration-tests) — blocked by a platform runtime regression where `capture:` variable scoping changed between SNAPSHOT releases (Nov 16–18). The playbooks are correct per documented API; the fix is upstream in `casehub-platform-yaml-step-runtime`.

After #288: #197 (ref-voice-tts-text-encoding), then Google/Gmail/Teams hardening cluster (#240, #192, #251, #249, #250, #243, #246, #253).

## Review Observations

- `listVoices` delegates to `VoiceRegistry.query()`, not `VoicePlatform` — if the generator uses `spi` to trace method chains rather than as a domain marker, the annotation may need revision. The issue specification explicitly requested `spi = VoicePlatform.class` for all three voice methods.

## References

- `graphql/src/main/java/io/casehub/connectors/graphql/ConnectorVoiceApi.java` — voice MCP tools with `@GeneratePlugin`
- `graphql/src/main/java/io/casehub/connectors/graphql/ConnectorMessagingApi.java` — messaging with `@GeneratePlugin(exclude = true)`
- `voice-spi/src/main/java/io/casehub/connectors/voice/spi/RoutingVoicePlatformService.java` — routing layer the scanner can't follow
