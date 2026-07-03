# Chat-Demo User Identity Design

**Issue:** #52
**Date:** 2026-07-03
**Status:** Approved
**Depends on:** casehub-pages#88 (dev-auth) — CLOSED

## Overview

Add per-user identity to the chat-demo module. Currently all messages are
sent with sender `"ref"` (the platform's hardcoded identity from
`RefChatPlatform.id()`). This feature replaces that with real JWT-based
authentication where each user picks or types a name, gets a signed JWT,
and all subsequent actions are attributed to that identity.

This is a refinement of section 1 of the interactive features spec
(`specs/2026-07-02-chat-demo-interactive-features-design.md`), now
concrete with dependency decisions and verified against the shipped
casehub-pages#88 implementation.

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

- **Inject `SecurityIdentity`** (Quarkus CDI) and **`ChatBackend`** directly
  (alongside existing `ChatPlatform`).
- **`postMessage()` and `postReply()` bypass the SPI** — call
  `chatBackend.storeMessage()` with `new MemberRef(identity.getPrincipal().getName())`
  as sender. They do NOT go through `ChatPlatform.messaging().send()` or
  `ChatPlatform.threading().reply()`.

  Rationale: the Messaging and Threading SPIs have no sender parameter —
  correctly, because real platforms (Slack, Discord, IRC) determine sender
  from credentials/tokens. Adding a sender parameter would be a dead
  parameter on every real implementation. The demo is not a client of an
  external platform; it IS the platform. Direct backend access is
  architecturally correct. `ChatBackend.storeMessage()` already accepts
  `MemberRef sender`.

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

If no presence entry exists for the identity (e.g. stale session or new
user), create it on-the-fly before processing the request.

### WebSocket Authentication

Client passes JWT as query parameter: `/ws/chat?token=<jwt>`. The
`@OnOpen` handler extracts the token from the query string and validates
it using `JsonWebToken` (injected via SmallRye JWT). If validation fails
or no token is present, the connection is accepted but treated as
unauthenticated (no identity associated). Standard practice — browser
WebSocket API cannot send custom headers.

### Identity Lifecycle Sequence

1. User picks or types name in login gate → `POST /dev/auth/login` → JWT
2. Client calls `PUT /api/presence/{memberId}` (status `ONLINE`) with Bearer token
3. `sessionStorage` stores the JWT
4. Every REST request sends `Authorization: Bearer <token>`
5. On message send: `ChatResource` reads identity from `SecurityIdentity`,
   validates channel exists (404 if not), checks membership (auto-adds if
   missing), calls `chatBackend.storeMessage()` with identity as sender
6. If identity has no presence entry, create it on-the-fly

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

## Non-Goals

- Per-user reaction tracking (who reacted)
- Keycloak or external OIDC provider (dev-auth uses self-issued JWTs)
- Passwords (login is name-only, JWT issued without password)
- Persistent identity across browser sessions (sessionStorage only)
- Thread grouping, collapsing, or side panels
