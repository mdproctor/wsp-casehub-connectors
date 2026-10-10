# Handover — casehub-connectors

## What Happened

Completed ConnectionPlatform epic (#147). Originally scoped as 8 child issues to build a new SPI, OAuth2 engine, and provider adapters. Brainstorming discovered platform was building the same infrastructure (#525/#526/#528/#530). Rescoped: platform owns ServiceConnectionProvider, ScopeRegistry, StaticCredentialStore. Connectors' work reduced to consuming platform types.

Migrated 6 modules (calendar-google, contacts-google, email-google, document-google, project-github, location-google) from ad-hoc credential resolvers to ServiceConnectionProvider/StaticCredentialStore. Added @RequiresScopes to all Google/GitHub consumers. Deleted 11 obsolete files. Landed as PR#160.

Designed ActionableNotification model for human-in-the-loop confirmation (platform#561) — notification that requires a response via pluggable ResponseChannels (SMS/WhatsApp/Signal via connectors, in-app via platform). Platform delivered #561.

## Decisions

- Tenancy: hardcoded "default" — multi-tenant support deferred until connectors has a tenancy story
- Credential resolvers: immediate replacement, not bridge adapters
- TrueLayer migration: deferred — keeps its own PSD2 consent lifecycle for now
- ActionableNotification extends Notification (not a WorkItem variant) — shared inbox via Inboxable interface

## Cross-Repo

- platform#560 — TokenRefreshClient wiring (open, blocks live token refresh)
- platform#561 — ActionableNotification + ResponseChannel (closed, landed)
- blocks-ui#232 — Connection status UI component (open, moved from connectors#159)
- neocortex#480 — SearchableProvider adapters for platform search SPIs (closed, landed)

## Next

- connectors#156 — Add @RequiresConfirmation to sensitive SPI methods, wire SMS/WhatsApp/Signal as ResponseChannels. Unblocked by platform#561.
