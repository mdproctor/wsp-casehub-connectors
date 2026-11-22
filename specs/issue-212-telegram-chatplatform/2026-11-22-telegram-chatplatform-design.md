# Design: Telegram ChatPlatform Provider — #212

## Summary

Add a Telegram ChatPlatform provider via Bot API, following the established two-module pattern. Provides 7 native capabilities (Messaging, Threading, Discovery, Reactions, Members, Presence, Commands) with webhook-based inbound and secret token verification.

## Modules

### `telegram-bot` — Shared HTTP Client

Thin HTTP client wrapping the Telegram Bot API. Uses `HttpHelper.CLIENT` (PP-20260607-9794cb). Token passed at call time, not injected (PP-20260609-0c3e24).

**Package:** `io.casehub.connectors.telegram.bot`

**`TelegramBotClient`** — `@ApplicationScoped` CDI bean. All public methods take `token` as first parameter. Base URL pattern: `https://api.telegram.org/bot{token}/{method}`.

| Method | Bot API method | Returns |
|--------|---------------|---------|
| `sendMessage(token, chatId, text, replyToMessageId)` | sendMessage | `TelegramMessage` |
| `setMessageReaction(token, chatId, messageId, emoji)` | setMessageReaction | `boolean` |
| `getChat(token, chatId)` | getChat | `TelegramChat` |
| `getChatMemberCount(token, chatId)` | getChatMemberCount | `int` |
| `getChatMember(token, chatId, userId)` | getChatMember | `TelegramChatMember` |
| `getChatAdministrators(token, chatId)` | getChatAdministrators | `List<TelegramChatMember>` |
| `setMyCommands(token, commands)` | setMyCommands | `boolean` |
| `deleteMyCommands(token)` | deleteMyCommands | `boolean` |
| `setWebhook(token, url, secretToken)` | setWebhook | `boolean` |
| `deleteWebhook(token)` | deleteWebhook | `boolean` |

Private helper methods: `post(token, method, body)`, `get(token, method)` following `SignalClient` patterns. JSON serialisation via Jackson `ObjectMapper`.

**Model records** in `io.casehub.connectors.telegram.bot.model`:

- `TelegramMessage(long messageId, TelegramChat chat, TelegramUser from, String text, long date, Long replyToMessageId)` — core message representation
- `TelegramChat(long id, String type, String title, String description, String firstName, String lastName, String username)` — group/channel/private chat
- `TelegramUser(long id, String firstName, String lastName, String username, boolean isBot)` — user identity
- `TelegramChatMember(TelegramUser user, String status)` — membership with status (creator/administrator/member/restricted/left/kicked)
- `TelegramUpdate(long updateId, TelegramMessage message, TelegramMessage editedMessage, TelegramCallbackQuery callbackQuery)` — webhook update payload
- `TelegramCallbackQuery(String id, TelegramUser from, TelegramMessage message, String data)` — inline keyboard callback
- `SendResponse(boolean ok, TelegramMessage result)` — API response wrapper

**Dependencies:**
- `casehub-connectors-api` (HttpHelper, model types)
- `quarkus-arc` (CDI)
- `jackson-databind` (JSON)
- Test: `junit-jupiter`, `assertj-core`, `wiremock-standalone`

### `chat-telegram` — ChatPlatform Implementation

**Package:** `io.casehub.connectors.chat.telegram`

#### `TelegramChatPlatform`

`@ApplicationScoped implements ChatPlatform`. Constructor-injected with `TelegramBotClient` and config properties.

**Config properties:**
- `casehub.telegram.bot.token` — Bot API token
- `casehub.telegram.webhook.secret-token` — webhook verification secret

**`@PostConstruct init()`**: If token is blank, calls `initDegraded()` (all capabilities return no-ops/empties). Otherwise sets `activeCapabilities` to the 7 native capabilities and wires up live implementations.

**Native capabilities (7):**

| Capability | Implementation |
|------------|---------------|
| `Messaging` | Lambda → `sendMessage`. Maps `ChatContent.text()` or `ChatContent.markdown()` to Telegram text. Returns `SendResult` with `ChatMessageRef(channel, String.valueOf(messageId))`. |
| `Threading` | Native reply threading via `reply_to_message_id` parameter on `sendMessage`. Wraps messaging lambda to pass `replyToMessageId` when a parent ref is provided. Falls back to `ChannelFallbackThreading` if no parent ref. |
| `Discovery` | Lambda → returns cached known chats. Telegram Bot API has no "list all chats" — bots only see chats they've received messages in. The inbound connector populates a concurrent cache of known chat IDs; discovery returns those chats via `getChat` lookups. |
| `Reactions` | Inner class `TelegramReactions implements Reactions`. `add()` calls `setMessageReaction`. `remove()` calls `setMessageReaction` with empty list. `list()` returns empty (Bot API does not expose reaction lists on messages). |
| `Presence` | Maps `TelegramChatMember.status` to presence: `creator`/`administrator`/`member` → online-equivalent, `left`/`kicked` → offline-equivalent, `restricted` → away-equivalent. Private chats use `getChat` for last-seen approximation. |
| `Members` | Lambda → `getChatAdministrators` for group/supergroup chats. `getChatMember` for individual lookups. Returns `List<Member>` mapped from `TelegramChatMember`. |
| `Commands` | Inner class `TelegramCommands implements Commands`. Registers commands via `setMyCommands` at startup. Command dispatch via `CommandService` (existing CDI infrastructure). |

**Degraded capabilities (3):**

| Capability | Fallback |
|------------|----------|
| `ChannelManagement` | `NoOpChannelManagement` — bots cannot create/delete groups via Bot API |
| `MemberManagement` | `NoOpMemberManagement` — bots have limited member management (can ban/unban but not add) |
| `MessageHistory` | `EmptyMessageHistory` — Bot API has no message history retrieval |

**`id()`** returns `"telegram"` (PP-20260609-e3a2bd).

**`supports(Class<?>)`** checks `activeCapabilities.contains(capability)`.

#### `TelegramInboundConnector`

`extends WebhookInboundConnector`. Receives Telegram webhook POSTs.

**`id()`** returns `InboundConnectorIds.TELEGRAM_INBOUND` (`"telegram-inbound"`).

**`handle(WebhookRequest request)`:**
1. Extract `X-Telegram-Bot-Api-Secret-Token` header from request
2. Compare against configured `casehub.telegram.webhook.secret-token`
3. If mismatch → return `WebhookResult.Unauthorized`
4. Parse JSON body as `TelegramUpdate`
5. Extract message from `update.message` (or `update.editedMessage`, `update.callbackQuery.message`)
6. If no message content → return `WebhookResult.Ignored`
7. Build `InboundMessage` with metadata (`telegram-chat-id`, `telegram-message-id`, `telegram-sender-id`, `telegram-chat-type`)
8. Update known-chats cache (for Discovery)
9. Return `WebhookResult.Delivered(inboundMessage)`

#### `TelegramInboundTranslator`

`implements InboundTranslator`. Translates `InboundMessage` → `ReceivedMessage`.

**`connectorType()`** returns `InboundConnectorTypes.TELEGRAM` (`"telegram"`).

**`translate(InboundMessage)`:** Maps metadata fields to `ChatChannelRef`, `ChatMessageRef`, `MemberRef`, `ChatContent`. Reply-to metadata maps to `parentRef` for threading.

#### `ChatTelegramBeans`

`@ApplicationScoped` CDI producer. Produces `TelegramInboundTranslator`.

**Dependencies:**
- `casehub-connectors-telegram-bot` (TelegramBotClient)
- `casehub-connectors-chat-spi` (ChatPlatform, capability interfaces)
- `casehub-connectors-core` (InboundConnector, WebhookInboundConnector)
- `quarkus-arc` (CDI)
- Test: `quarkus-junit`, `junit-jupiter`, `assertj-core`, `wiremock-standalone`

## Constants (in `connectors-api`)

Add to `InboundConnectorIds`:
```java
public static final String TELEGRAM_INBOUND = "telegram-inbound";
```

Add to `InboundConnectorTypes`:
```java
public static final String TELEGRAM = "telegram";
```

## Discovery Cache Strategy

Telegram Bot API has no "list chats" endpoint. Bots are passive — they only learn about chats when they receive messages. The design uses a simple in-memory `ConcurrentHashMap<String, TelegramChat>` populated by the inbound connector on each received update. `Discovery.listChannels()` returns the cache contents via `getChat` refresh calls.

This is a known limitation — not a bug. Signal has a similar constraint (contacts are only known if the bot has interacted with them). The cache is ephemeral (lost on restart) which is acceptable for the initial implementation. A persistent cache can be added later if needed.

## Testing Strategy

**`telegram-bot` unit tests:**
- WireMock-backed tests for each `TelegramBotClient` method
- Verify correct URL construction (token in path), request body, response parsing
- Error handling: non-2xx responses, malformed JSON, timeouts

**`chat-telegram` unit tests:**
- `TelegramChatPlatform` capability wiring (active vs degraded)
- `TelegramInboundConnector` webhook handling: valid secret → Delivered, invalid secret → Unauthorized, missing message → Ignored
- `TelegramInboundTranslator` mapping correctness
- Known-chats cache population via inbound messages

**Integration tests:**
- Round-trip: inbound webhook → translator → ReceivedMessage with correct refs
- Degraded mode: missing config → all capabilities return no-ops

## Known Limitations

1. **No Discovery without inbound traffic** — bot cannot enumerate its chats. Discovery returns only chats seen via inbound messages.
2. **Reactions.list() returns empty** — Bot API does not expose reaction lists on messages. `add()` and `remove()` work.
3. **No ChannelManagement** — bots cannot create/delete groups or channels via Bot API.
4. **No MemberManagement** — bots can ban/unban users but the Bot API's member management doesn't map cleanly to the add/remove SPI.
5. **No MessageHistory** — Bot API has no message retrieval endpoint.
6. **Rate limiting** — Telegram enforces ~30 messages/second for bulk sending. The client does not implement rate limiting; callers must handle 429 responses.

## References

- [Telegram Bot API](https://core.telegram.org/bots/api) — API specification
- `signal-cli/SignalClient.java` — HTTP client pattern reference
- `chat-signal/SignalChatPlatform.java` — ChatPlatform implementation pattern (6 native capabilities)
- `chat-discord/DiscordChatPlatform.java` — ChatPlatform implementation pattern (9 native capabilities)
- `chat-discord/DiscordInboundConnector.java` — WebhookInboundConnector pattern
- `connectors-api/WebhookInboundConnector.java` — inbound SPI
- `docs/protocols/connectors/shared-http-client.md` (PP-20260607-9794cb) — HttpHelper.CLIENT requirement
- `docs/protocols/connectors/credential-config-ownership.md` (PP-20260609-0c3e24) — call-time credentials
- `docs/protocols/connectors/inbound-connector-id-constants.md` (PP-20260607-d4ee52) — ID constants requirement
- `docs/protocols/connectors/spi-id-method-naming.md` (PP-20260609-e3a2bd) — id() naming convention
- `docs/specs/2026-11-13-connector-gap-analysis.md` — Telegram identified as #2 n8n integration (951.6K deploys)
