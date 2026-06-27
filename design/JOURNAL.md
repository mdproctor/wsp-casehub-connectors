# Design Journal — issue-26-demo-chat-service

### 2026-06-28 · §4 · Solution Strategy — Layer Taxonomy

Extended the layer taxonomy with L7 (Chat Platform SPI extensions) and
L8 (Chat Demo). L7 adds three new capability interfaces
(ChannelManagement, MemberManagement, MessageHistory) and extends two
existing ones (Reactions with list(), Presence with set()). This follows
the composed-capabilities-with-auto-degradation pattern established in
the original ChatPlatform SPI — each new capability has a named degraded
type, the Builder auto-fills defaults, and supports() uses class tokens.

L8 is the demo chat service — a profile-gated (-Pdemo) Quarkus
application with SqliteChatBackend and REST endpoints that map 1:1 to
ChatPlatform capabilities. It is not published to GitHub Packages.

### 2026-06-28 · §5 · Building Block View — Module Structure

Added three modules to the module structure: chat-spi (extended with new
interfaces), chat-ref (gains ChatBackend interface and
InMemoryChatBackend), and chat-demo (new module, profile-gated). chat-ref
now produces a test-jar so chat-demo can reuse the ChatBackendContract
test class.

### 2026-06-28 · §10 · ChatBackend is internal, not a public SPI

ChatBackend (the storage interface for the reference ChatPlatform
implementation) lives in chat-ref, not in chat-spi. This was a deliberate
decision: IrcChatPlatform delegates to IrcClient, a future
SlackChatPlatform would delegate to SlackBotClient, and RefChatPlatform
delegates to ChatBackend. Each implementation owns its storage. The
persistence mechanism is not a platform-wide concern — it is internal to
the reference implementation.

InMemoryChatBackend is @DefaultBean (connectors CDI pattern), overridden
by SqliteChatBackend (@ApplicationScoped) when chat-demo is on the
classpath. This follows the existing connectors convention
(NoOpConnectorMeshBridge, DefaultEmailInboundAccountProvider) rather than
the platform-wide @Alternative @Priority ladder.

### 2026-06-28 · §10 · Channel record gains description field

Channel record extended from (ref, name, topic, isPrivate) to (ref, name,
topic, description, isPrivate). Topic is dynamic ("Sprint 47 standup"),
description is static ("Engineering general channel"). Slack and Discord
distinguish them; IRC has topic only (passes null for description). This
is a breaking change to the record constructor — all call sites updated.

### 2026-06-28 · §10 · Presence is no longer a functional interface

Adding set() to Presence breaks its single-abstract-method status.
External consumers using Presence as a lambda must switch to anonymous
classes. This is intentional — the degradation model requires every
method to be explicit in both native and degraded implementations.
