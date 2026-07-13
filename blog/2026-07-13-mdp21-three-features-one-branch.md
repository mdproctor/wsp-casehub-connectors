---
title: Three Features, One Branch
date: 2026-07-13
sequence: mdp21
tags: [chat-demo, progressive-disclosure, emoji, swipe, web-components]
---

Batched three chat-demo UI features onto a single branch: progressive
disclosure on messages (#63), an emoji reaction palette (#64), and
swipe-to-reveal gestures for phone drawers (#55). All Phase 2/3 from the
qhorus chat UI spec.

The progressive disclosure piece changed how messages work at a fundamental
level. Clicking a message used to immediately set it as the reply target —
a Phase 1 bootstrap that assumed every click meant "I want to respond to
this." That's wrong once you have an expanded view with correlation context,
artefact details, and commitment state. Now clicking toggles a dedicated
expand button, and reply is an explicit action inside the expanded view.
Two clicks instead of one, but the second click is informed by context the
first click reveals.

The design review caught several things I wouldn't have: the emoji picker
needs the Popover API to escape the feed's `overflow: hidden` clipping (a
`position: absolute` picker would be invisible), and the swipe controller
needs a snap handoff protocol to avoid a visual rebound when transitioning
from inline `transform` to CSS classes. Both were non-obvious failure modes
that only surface in specific viewport states.

The swipe controller also surfaced a jsdom gotcha — synthetic PointerEvents
have sub-millisecond timestamp deltas, so any velocity calculation guarded
by `dt > 0` produces absurd values. Fixed with a 5ms minimum window.
Submitted to the garden as GE-20260713-b35869.

`emoji-picker-element` was the right call. 15KB for a comprehensive,
accessible, Lit-native picker with search and categories. Building a
curated grid would have been less code but more maintenance, and the
component boundary is identical either way.

All three features are independently testable — progressive disclosure
in message tests, emoji picker in reaction bar tests, swipe in controller
tests. The workbench only coordinates where it must: passing `channelName`
down to messages, and hosting the SwipeController in phone mode.
