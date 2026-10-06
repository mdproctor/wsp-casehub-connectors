# Handover — casehub-connectors

## What Happened

Completed #141 (ref CDI normalisation) and #145 (Spring generator wiring) in one session. Investigated the Spring/Quarkus duality — discovered the ref implementations had no Spring auto-configuration. Wired the `spring-generator` plugin into `connectors-spring`, generating auto-configs for all 9 platform SPI refs. Filed neocortex#437 to generalize the search normalization pipeline beyond location, and updated connectors#140 to depend on it.

## Decisions

- Extend existing `connectors-spring` module rather than creating a new one — matches platform pattern of one `-spring` module per repo
- `SpringVerifyMojo` for verification (no runtime Spring integration test) — mechanical generation doesn't warrant it
- Backend interfaces made public (commerce, contacts, location) — required for generated cross-package auto-configs
- Connectors #140 (NLP normalization) narrowed to "adopt neocortex infrastructure" — blocked by neocortex#437

## Key Artifacts

- Design spec: `specs/issue-145-wire-spring-generator-for-ref-implementations/2026-10-06-spring-generator-ref-wiring-design.md`
- Plan: `plans/2026-10-06-spring-generator-ref-wiring.md`
- Manual config: `connectors-spring/src/main/java/io/casehub/connectors/spring/ProjectRefManualConfig.java`
- Contributor guide update: `docs/guides/contributor-guide.md` (§ Ref Implementation Seed Data)

## Next

- #144 — add search method to EmailPlatform SPI (small, identified in #139 audit)
