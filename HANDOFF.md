# HANDOFF — casehub-connectors

## Last Session

Design session for #188 (real-time inbound channels). Reframed scope after discovering WhatsApp inbound and email inbound already exist — actual gaps are media download, Gmail push, and VoicePlatform SPI. 12 design decisions captured and reviewed (3-round adversarial review revised 6 decisions). Spec written and reviewed (3-round standard). Implementation plan written: 4 batches, 7 tasks. Filed platform#680 (streaming step type) — it landed within the session, so the plan uses the shipping `StreamingStep<I, O>` API. Blog entry written: "Your Workflow Engine Is a Data Pipeline."

## Immediate Next Step

`work continue` → invoke `executing-plans` against `plans/2026-10-30-real-time-inbound-channels.md`. Start with Batch 1 (voice-spi + voice-ref). **Prerequisite for Batch 4 (Task 7):** agency must be on main and an IntelliJ workspace spanning both connectors and agency must be open. Use IntelliJ refactor-move for the cross-repo SPI adapter — move whole modules/packages, not individual classes.

## Cross-Module

- agency/speech-sherpa needs `casehub-connectors-voice-spi` dependency + `SherpaVoicePlatform` adapter (Task 7 in plan). Agency must be on main before starting this task.

## References

- `specs/issue-188-real-time-inbound-channels/2026-10-30-real-time-inbound-channels-design.md` — approved design spec
- `specs/issue-188-real-time-inbound-channels/decisions.md` — 12 validated decisions
- `plans/2026-10-30-real-time-inbound-channels.md` — implementation plan (4 batches, 7 tasks)
- `JOURNAL.md` — design session notes
- `docs/blog/2026-10-30-mdp01-your-workflow-engine-is-a-data-pipeline.md` — blog article
