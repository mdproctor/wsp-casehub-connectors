# Handover — casehub-connectors

## What Happened

Designed and implemented the Commands capability (#32) — full lifecycle
from brainstorming through merged PR.

**#32 (Commands capability):** New `Commands` capability on ChatPlatform
SPI. Cross-platform slash command registration, invocation dispatch, and
immediate + deferred response delivery. 7 decisions captured, spec
written and self-reviewed, 4 batches implemented (SPI foundation,
Discord, Ref, Slack+docs). Code review clean, branch audit clean across
all 4 dimensions. Squashed 9 → 4 commits, PR #112 merged to main.

Key deliverables:
- `CommandHandler` SPI + `CommandService` (CDI discovery + dispatch)
- `DiscordInteractionEndpoint` (Ed25519 verification)
- `SlackCommandEndpoint` (HMAC-SHA256 verification)
- `RefCommands` (in-memory)
- Garden revision: ManagedExecutor recursive constructor caveat

Created issues for next work: #113 (email-ref), #114 (email-google).

## What's Next

**#113 (RefEmailPlatform)** — in-memory EmailPlatform following the
established ref pattern. XS/S scale. Then #114 (GoogleEmailPlatform —
Gmail API provider, mirrors calendar-google pattern).

## Key Artifacts

- #32 design spec: `docs/specs/issue-32-discord-slash-commands/2026-09-27-commands-capability-design.md`
- #32 decisions: `docs/specs/issue-32-discord-slash-commands/decisions.md`
- #32 plan: `plans/2026-09-27-commands-capability.md`
- Garden revision: `~/.hortora/garden/jvm/GE-20260602-c4a68a.md` (recursive constructor caveat)
