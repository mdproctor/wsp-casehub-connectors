# Making the Demo Talk Back

The chat-demo shipped last session as a viewer — messages scrolled by, channels
appeared in the sidebar, presence dots blinked green. Pretty, but inert. You
couldn't reply, react, create a channel, or delete one. The WebSocket protocol
pushed data at you and you watched it arrive.

That changed today. Three features turned the demo from a dashboard into
something you can actually use: channel management (create and delete), emoji
reactions, and threaded replies.

## The SPI question

Channel deletion was the interesting one architecturally. `ChannelManagement`
had `create()` and `find()` — no `delete()`. Adding it meant updating every
implementation: the reference backend, Slack, Discord, IRC's no-op fallback.
The protocol says all implementations update in the same commit, so that's what
happened — five implementations, one commit, no intermediate broken state.

The platform differences surfaced immediately. Discord has a real
`DELETE /channels/{id}` endpoint. Slack doesn't delete channels at all — it
archives them via `conversations.archive`. The SPI method is called `delete()`
because that's what the caller means; what the platform does underneath is its
own business. The abstraction earns its keep here.

The cascade was more mechanical but had to be right. Deleting a channel means
deleting its reactions, messages, members, then the channel itself — in that
order, in a transaction. The SQLite backend wraps it in `setAutoCommit(false)`
with rollback on failure. The in-memory backend just removes from four maps.
Same contract test covers both.

## Reactions and the missing snapshot

Reactions had a gap nobody noticed until now: the WebSocket `buildSnapshot()`
sent channels, messages, members, and presence on connect — but not reactions.
The `REACTION_COLUMNS` were defined, `broadcastReactionAppend()` existed, but
a client connecting after reactions were added would see empty pills. Adding
reactions to the snapshot fixed it. The remove broadcast was also missing — you
could add a reaction and see it appear on all clients, but removing one was
silent.

The frontend emoji palette is deliberately minimal: six Unicode emojis in a
popup, click to toggle. No custom emoji, no per-user tracking, no Slack-style
`:shortcode:` support. It's a demo. The palette dismisses on selection or
outside click — the standard interaction pattern, nothing clever.

## Discord-style replies

Threading was a UI design decision more than a backend one. The data model
already had `parentId` on messages. The SPI's `Threading.reply()` already
worked. The question was how to render it.

Slack's model — threads hidden in a side panel with "N replies" under the
parent — is powerful but complex. Discord's model — replies inline with a small
reference bar pointing to the parent — is simpler and reads better at a glance.
For a demo, the second choice was obvious. Click the reference bar and the
parent scrolls into view with a brief highlight animation.

The inter-panel coordination was the most detailed part. When you click the
reply icon on a message, `message-list` emits a `reply-to` event.
`message-input` picks it up, shows a "Replying to **SenderName**" banner, and
routes the submit to the `/replies` endpoint instead of `/messages`. Cancel
clears the state. Channel switch clears it too — the target message isn't
visible any more. No bidirectional state to manage; the message list doesn't
need to know whether the input is in reply mode.

## What's still missing

Every message shows sender `"ref"` — the hardcoded platform identity. Real
usernames need authentication, and that's moving to casehub-pages as a
platform-wide dev-auth module using SmallRye JWT. Once that lands, the demo
gets a login gate, identity switching, and messages attributed to real names.
The plumbing is ready — `ChatBackend.storeMessage()` already takes a
`MemberRef sender`. It's just waiting for the identity to come from somewhere
real instead of being hardcoded.
