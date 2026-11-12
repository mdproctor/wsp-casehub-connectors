---
layout: post
title: "When the Agent Picks Up the Phone"
date: 2026-11-12
entry_type: article
subtype: diary
projects: [casehubio/connectors]
tags: [mobile, telephony, spi, voice-calls, agent, architecture]
series: issue-206-mobile-platform
---

# When the Agent Picks Up the Phone

Every connector SPI we've built so far assumes the interaction is discrete. Send a message, get a response. Query an API, get data back. Even the voice platform — transcribe some audio, synthesise some speech — is request/response at its core. No state between calls, no conversation that outlives a single method invocation.

Telephony breaks that model. A phone call is a sustained, stateful interaction where the agent needs to listen, think, and respond in real time — potentially for minutes. It needs to negotiate ("we don't have 7pm, would 7:30 work?"), make decisions mid-conversation, and sometimes reach out to a human for approval before continuing. A fire-and-forget connector can't do this. A capability-based platform SPI can.

## The Instruction-Based Model

The conventional approach to programmable voice is bidirectional audio streaming — your code receives raw audio bytes, runs speech-to-text, decides what to say, generates speech, sends audio bytes back. You own the entire conversation pipeline. It's powerful and it's a lot of plumbing.

Modern telephony APIs like [Dial](https://docs.getdial.ai) offer something more interesting: instruction-based calls. You provide a goal — "call this restaurant and book a table for 2 at 7pm, name is Smith" — and the provider's AI handles the entire conversation. It says hello, explains the booking, negotiates availability, confirms details, and hangs up. Your agent gets back a structured transcript with speaker turns and timing.

```java
Call call = platform.voiceCall().initiate(
    "+447700900000",
    "Book a table for 2 at 7pm tonight for Smith. "
    + "Be polite. If 7pm is unavailable, accept any time within an hour.");
// call.transcript() → the full conversation
// call.status() → COMPLETED, FAILED, BUSY, NO_ANSWER
```

The agent doesn't manage TTS or STT. It doesn't handle turn-taking. It sets a goal and reads the outcome. For straightforward interactions — bookings, enquiries, confirmations — this is the right abstraction. The interesting complexity is in what goal you set, not in how you handle audio codec buffering.

## When Goals Aren't Enough

But sometimes the agent needs to be in the conversation, not just set it up. The restaurant says they only have 8pm. The agent needs to check with the person who asked for the booking — "they only have 8pm, is that OK?" — before committing. That means pausing the live phone call, consulting a human through a work queue or chat message, waiting for a response, then resuming: "yes, 8pm works."

This is where `ConversationHandler` comes in. Instead of a string instruction, the caller provides a callback interface that receives the human's speech as text and returns what to say next:

```java
interface ConversationHandler {
    ConversationResponse onHumanSpeech(
        String text, List<TranscriptTurn> fullTranscript);
}
```

The provider handles audio — Dial through its self-hosted LLM WebSocket protocol, Twilio through Media Streams with platform STT/TTS. The handler operates entirely at the text level. It can take as long as it needs before responding — check a database, call an API, ask a human for approval. The call waits.

The response carries an optional action — `Hold` (speak some text, then immediately give the handler another turn without waiting for the human), `Transfer`, or `HangUp`:

```java
return new ConversationResponse(
    "Please hold while I check availability",
    new ConversationAction.Hold());
```

![Two voice call modes — provider-managed and platform-managed](images/mobile-call-modes.svg)

Both modes present as text at the SPI surface. Audio handling — WebSocket protocols, codec conversion, STT/TTS pipelines — is internal to each provider implementation. The consumer never touches audio.

## Five Domains, Six Steps

The dinner booking playbook is the proof that platform SPIs compose. Location finds the restaurant. Mobile calls it. Email confirms. SMS notifies. Chat reports back.

```yaml
steps:
  - location-search-nearby:
      latitude: 51.5074
      longitude: -0.1278
      category: restaurant
      keyword: thai

  - location-place-details:
      placeId: ${restaurants.items[0].placeId}

  - mobile-call:
      to: ${restaurant.phone}
      instruction: >
        Book a table for 2 at 7pm tonight for Smith.
        If 7pm is unavailable, accept within an hour.

  - mobile-send-sms:
      to: "+447700900000"
      body: "Booking confirmed at ${restaurant.name}."

  - messaging-send-email:
      to: bookings@thai-garden.example.com
      subject: "Booking confirmation — Smith, 2 guests"

  - chat-send:
      channel: personal
      message: "Dinner sorted. ${restaurant.name}, 7pm."
```

![Dinner booking flow — five domains, six steps](images/dinner-booking-flow.svg)

Each step is a different connector platform. Each platform has a reference implementation for testing — no API keys, no network, deterministic results in CI. The voice call step matches the instruction against canned conversation scripts in seed data and returns a pre-built transcript. Same SPI interface, same playbook syntax, real or simulated backends selected by deployment.

## Why Telephony Needs a Platform SPI

The old `TwilioSmsConnector` in the codebase was a fire-and-forget connector — `send(to, body)` and done. No capability introspection, no provider abstraction, no graceful degradation. Adding voice calls, voicemail, WhatsApp messaging, and number management to that model would have produced a bag of unrelated methods on a monolithic interface.

`MobilePlatform` follows the same composition pattern as `ChatPlatform` — a Builder assembles capabilities, unsupported ones get NoOp degradation defaults, `supports()` tells you what's real. Six capabilities: `Sms`, `VoiceCall`, `VoiceCallControl`, `Voicemail`, `Messaging`, `NumberInfo`. Each provider declares what it supports. Dial gets all six. A future SMS-only provider gets one. The SPI doesn't care.

The SPI, reference implementation, and MCP tools are working. What remains is the production providers — Dial for instruction-based calls and SMS, Twilio for the full audio pipeline — and the legacy connector absorption that consolidates SMS and WhatsApp delivery into the new platform pattern.
