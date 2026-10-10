# Journal — casehub-connectors

## 2026-10-06 → 2026-10-10 — ConnectionPlatform epic (#147)

Started with an 8-issue epic to build a ConnectionPlatform SPI for unified authentication. Brainstorming surfaced that platform was independently building the same infrastructure (#525/#526/#528/#530). Rescoped dramatically: platform owns the OAuth2 engine, ServiceConnectionProvider SPI, ScopeRegistry, and StaticCredentialStore. Connectors' role reduced to consuming platform's types.

Filed platform#548 with full API specifications. Platform delivered everything (8 commits for #548, 7 for #530). Connectors work became mechanical migration: replace 6 ad-hoc credential resolvers with ServiceConnectionProvider/StaticCredentialStore, add @RequiresScopes to all Google/GitHub consumers, delete 11 obsolete files. Landed as PR#160.

Closed 12 issues (epic #147 + children #148-155 as superseded, #157/#158 as completed, #159 moved to blocks-ui#232). Closed #140 after confirming neocortex#480 delivered the NLP normalization work.

Designed the ActionableNotification model for human-in-the-loop confirmation of sensitive operations (platform#561) — evolved through 4 iterations: specialised WorkItem → InboxItem base → standalone Approval → ActionableNotification extending Notification with pluggable ResponseChannels. Platform delivered #561.

Next: connectors#156 — add @RequiresConfirmation to sensitive SPI methods, wire SMS/WhatsApp/Signal as response channels.
