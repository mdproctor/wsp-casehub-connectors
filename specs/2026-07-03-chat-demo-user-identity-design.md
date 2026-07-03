# Chat-Demo User Identity Design

**Issue:** #52
**Date:** 2026-07-03
**Status:** Approved
**Depends on:** casehub-pages#88 (dev-auth) — CLOSED
**Supersedes:** Section 1 of `specs/2026-07-02-chat-demo-interactive-features-design.md`

## Overview

Add per-user identity to the chat-demo module. Currently all messages are
sent with sender `"ref"` (the platform's hardcoded identity from
`RefChatPlatform.id()`). This feature replaces that with real JWT-based
authentication where each user picks or types a name, gets a signed JWT,
and all subsequent actions are attributed to that identity.

This spec supersedes section 1 of the interactive features spec
(`specs/2026-07-02-chat-demo-interactive-features-design.md`). Section 1
of that spec should be updated with a forward reference to this document.
This spec is the authoritative design for user identity in chat-demo.

---

## Architecture

### Dependency: casehub-pages-auth

Chat-demo adds `io.casehub:casehub-pages-auth:0.1-SNAPSHOT` as a Maven
dependency. This brings in:

- `DevAuthResource` — JAX-RS `POST /dev/auth/login` endpoint, profile-gated
  with `@UnlessBuildProfile("prod")`, auto-discovered via CDI + Jandex index
- `quarkus-smallrye-jwt` (transitive) — enables `SecurityIdentity` injection
  and JWT validation
- `smallrye-jwt-build` (transitive) — for JWT signing
- `LoginRequest` / `TokenResponse` records

No code duplication. The endpoint, records, and profile gating come from
the dependency.

**Cross-repo dependency note:** This creates a new dependency path:
`casehub-connectors` (chat-demo module) → `casehub-pages` (auth module).
This dependency is scoped to the demo profile (`-Pdemo`) and the chat-demo
module is unpublished (`skipDeploy=true`, `maven.deploy.skip=true`). The
`casehub-connectors (no casehubio deps)` constraint in PLATFORM.md applies
to published artifacts; chat-demo is not published.

Required documentation updates:
- **ARC42STORIES.MD L8** — add `casehub-pages-auth` to chat-demo dependencies
- **PLATFORM.md Cross-Repo Dependency Map** — add row:
  `casehub-pages-auth` → `casehub-connectors` / `chat-demo` / profile-gated
  demo dependency (unpublished module)

### Data Flow

1. Browser → `POST /dev/auth/login` with `{ "name": "alice" }` → signed JWT
2. JWT stored in `sessionStorage` under key `pages-dev-auth-token`
3. All REST requests include `Authorization: Bearer <token>`
4. Quarkus validates JWT signature, populates `SecurityIdentity`
5. `ChatResource` reads `identity.getPrincipal().getName()`
6. Calls `ChatBackend.storeMessage()` directly with user's identity as sender

---

## Backend Changes

### ChatResource

- **Add `@Authenticated` at class level** — all endpoints require a valid
  JWT. Without this annotation, `SecurityIdentity` is injectable but
  represents an anonymous principal, and `getPrincipal().getName()` returns
  `"anonymous"` rather than rejecting the request. Unauthenticated requests
  receive HTTP 401, which triggers the frontend's `pages-auth-expired`
  handler to re-show the login gate.

- **Inject `SecurityIdentity`** (Quarkus CDI) and **`ChatBackend`** directly
  (alongside existing `ChatPlatform`).

- **`postMessage()` and `postReply()` bypass the SPI** — call
  `chatBackend.storeMessage()` with all five parameters:
  1. `platformId` = `"ref"` — the demo uses the same logical platform as
     `RefChatPlatform`. Messages from `ChatResource` and `RefChatPlatform`
     are stored in the same `ChatBackend` and should share platform identity.
  2. `channel` = the `ChatChannelRef` from the path parameter
  3. `content` = `new ChatContent(request.text())`
  4. `sender` = `new MemberRef(identity.getPrincipal().getName())`
  5. `parentRef` = `null` for `postMessage()`, the parent `ChatMessageRef`
     for `postReply()`

  They do NOT go through `ChatPlatform.messaging().send()` or
  `ChatPlatform.threading().reply()`.

  Rationale: the Messaging and Threading SPIs have no sender parameter —
  `Messaging.send(ChatChannelRef, ChatContent)` and
  `Threading.reply(ChatMessageRef, ChatContent)` — correctly, because real
  platforms (Slack, Discord, IRC) determine sender from credentials/tokens.
  Adding a sender parameter would be a dead parameter on every real
  implementation. The demo is not a client of an external platform; it IS
  the platform. Direct backend access is architecturally correct.
  `ChatBackend.storeMessage()` already accepts `MemberRef sender`.

  `ChatResource` injects both `ChatPlatform` (for reactions, presence,
  members, channels, discovery, history) and `ChatBackend` (for
  send/reply with caller-specified identity).

### Auto-Membership

On message send, if the identity is not already a member of the target
channel, `ChatResource` auto-adds them via
`ChatPlatform.memberManagement().add()` before storing the message. The
`displayName` is the principal name from the JWT. Auto-membership also
broadcasts the new member via `broadcaster.broadcastMemberAppend()`.

### Presence Auto-Create

On message send, `ChatResource` checks the sender's presence via
`ChatPlatform.presence().of(memberRef)`. If the result is
`PresenceStatus.UNKNOWN` (the default returned by `InMemoryChatBackend`
when no entry exists):

1. Set presence to `PresenceStatus.ONLINE` via
   `ChatPlatform.presence().set(memberRef, PresenceStatus.ONLINE)`
2. Broadcast the presence change via
   `broadcaster.broadcastPresenceReplace(memberRef, PresenceStatus.ONLINE)`

This is idempotent: if the entry already exists with any status other
than `UNKNOWN`, no auto-creation occurs. A user who explicitly set
themselves `OFFLINE` will not be overridden — only the `UNKNOWN`
(never-seen) state triggers auto-creation.

This check runs in `ChatResource.postMessage()` and
`ChatResource.postReply()` only — not in a filter or interceptor.
Presence creation for the normal login flow happens explicitly via
`PUT /api/presence/{memberId}` with status `ONLINE`.

### WebSocket Authentication

Client passes JWT as query parameter: `/ws/chat?token=<jwt>`.

**Mechanism:** A `@PreMatching ContainerRequestFilter` reads the `token`
query parameter from the HTTP upgrade request and sets it as an
`Authorization: Bearer <token>` header. SmallRye JWT then processes it
transparently through the standard Quarkus security pipeline, populating
`SecurityIdentity` on the WebSocket connection.

The `@WebSocket` endpoint is annotated `@Authenticated`. Connections
without a valid JWT are rejected at upgrade time with HTTP 401. There is
no use case for unauthenticated WebSocket connections — the login gate
blocks all UI interaction until login completes, and the WebSocket
connects (or reconnects) only after successful authentication.

The `ContainerRequestFilter` is scoped to WebSocket upgrade requests
only (check for `Upgrade: websocket` header) to avoid interfering with
REST endpoints that already receive the token in the `Authorization`
header.

### Identity Lifecycle Sequence

1. User picks or types name in login gate → `POST /dev/auth/login` → JWT
2. Client calls `PUT /api/presence/{memberId}` (status `ONLINE`) with Bearer token
3. `sessionStorage` stores the JWT
4. WebSocket connects with token: `/ws/chat?token=<jwt>`
5. Every REST request sends `Authorization: Bearer <token>`
6. On message send: `ChatResource` reads identity from `SecurityIdentity`,
   validates channel exists (404 if not), checks membership (auto-adds if
   missing), auto-creates presence if `UNKNOWN`, calls
   `chatBackend.storeMessage("ref", channel, content, sender, parentRef)`

---

## Frontend Changes

### Login Gate

Add `<pages-dev-auth>` (from `@casehubio/pages-ui`) to the app shell.
Attributes:
- `backend-url=""` — same origin, empty string
- `identities="..."` — populated from the `members` dataset, deduplicated
  by memberId from the WebSocket snapshot

On first load with no JWT in `sessionStorage`, the overlay blocks
interaction until login completes. The component handles token expiry
detection and re-shows the gate when `pages-auth-expired` is dispatched.

### Identity Widget

Add `<pages-identity>` (from `@casehubio/pages-ui`) in the message input
area. Click opens a popover picker to switch identity. On switch:

1. Calls `POST /dev/auth/login` with new name → new JWT
2. Updates `sessionStorage`
3. Sends presence `OFFLINE` for old identity, `ONLINE` for new one via
   `PUT /api/presence/{memberId}`
4. All subsequent messages use the new identity

### Authorization Header

All REST calls include `Authorization: Bearer <token>` read from
`sessionStorage` key `pages-dev-auth-token`. Applies to message send,
reply, reactions, channel operations — everything.

### WebSocket Reconnect

On login or identity switch, reconnect the WebSocket with the new token
as query parameter. Connection URL: `/ws/chat?token=<jwt>`.

**Reconnect ordering:** close the old WebSocket connection first, then
open the new connection with the updated token. During the brief gap
(typically milliseconds), no real-time events are received. This is
acceptable — the new connection's `@OnOpen` handler delivers a fresh
snapshot containing all messages, so no data is lost. Messages that
arrived during the gap are present in the snapshot; the user simply
does not see them arrive in real-time.

### Token Expiry Handling

If a REST call returns 401, dispatch `pages-auth-expired` custom event.
The `<pages-dev-auth>` component listens for this and re-shows the login
gate.

---

## Maven Changes

**chat-demo/pom.xml** — add dependency:
```xml
<dependency>
    <groupId>io.casehub</groupId>
    <artifactId>casehub-pages-auth</artifactId>
    <version>0.1-SNAPSHOT</version>
</dependency>
```

This requires the casehub-pages GitHub Packages repository to be
configured (already present in the parent POM).

---

## Known Limitations

- **Identity uniqueness:** The dev-auth login is name-only with no
  password. Two users who type the same name (e.g. "alice") in different
  browser tabs receive separate JWTs but with the same `sub` claim,
  producing the same `MemberRef`. They share presence, membership, and
  message attribution. Setting presence `OFFLINE` for "alice" in tab 2
  also affects tab 1's view of alice's presence. This is an inherent
  property of passwordless dev-auth and mirrors IRC-style identity
  semantics. For a demo, this is acceptable — the login gate provides
  name selection, not account security.

- **Identity = display name:** `MemberRef.id` serves as both the unique
  identifier and the display name (via `Member.displayName`). There is no
  separate username/display-name distinction. This is intentional for
  demo simplicity — the name typed in the login gate is what appears in
  messages.

## Non-Goals

- Per-user reaction tracking (who reacted)
- Keycloak or external OIDC provider (dev-auth uses self-issued JWTs)
- Passwords (login is name-only, JWT issued without password)
- Persistent identity across browser sessions (sessionStorage only)
- Thread grouping, collapsing, or side panels
