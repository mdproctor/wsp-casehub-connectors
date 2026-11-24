---
layout: post
title: "Real SIMs for Real Testing"
date: 2026-11-24
entry_type: note
subtype: diary
projects: [casehubio/connectors]
tags: [mobile, sms, testing, numbrrs, webhook]
---

# Real SIMs for Real Testing

Signal rejects VoIP numbers. So does Telegram. WhatsApp is the same. If you want to verify a phone number on any of these platforms, you need a real SIM — an actual mobile number registered on a physical network. VoIP providers like Twilio give you programmable numbers that are excellent for sending and receiving SMS, but the moment you try to register one with a messaging app, it fails silently or gets flagged.

This is the problem Numbrrs solves. It's a small UK provider that gives you real mobile SIM numbers — £1.50 a month each — with an API for receiving SMS and voice call transcriptions. You can't send outbound SMS through them. What you can do is point Signal, Telegram, or WhatsApp at one of these numbers, receive the verification code via webhook, and have a real registered identity on each platform.

## The inbound-only provider

The `mobile-numbrrs` module wraps Numbrrs as a MobilePlatform provider, but unlike Twilio or Dial it only supports two of the six capabilities: `Sms` (read-only) and `NumberInfo`. The `Sms.send()` method returns `SendResult.failure()` immediately — Numbrrs simply doesn't do outbound. The `listMessages()` method pulls received messages from their REST API, and `NumberInfo` returns the configured numbers with their capabilities.

This asymmetry is deliberate and the SPI handles it cleanly. The `MobilePlatform.builder()` pattern means you register only the capabilities you support — everything else degrades gracefully via the `NoOpMobilePlatform` defaults, and `supports(Class<?>)` tells callers exactly what's available before they try it.

```java
this.delegate = MobilePlatform.builder("numbrrs")
        .sms(new NumbrrsSms(client))
        .numberInfo(new NumbrrsNumberInfo(numbers))
        .build();
```

Two lines of wiring. The builder handles the rest — `supports(VoiceCall.class)` returns false, `voiceCall()` throws `UnsupportedCapabilityException` with recovery metadata for LLM self-correction. No defensive code needed in the provider.

## Webhook security

The inbound connector follows the same pattern as every other webhook connector in the platform: extend `WebhookInboundConnector`, verify the signature, parse the payload, return a `WebhookResult`. Numbrrs signs webhooks with HMAC-SHA256 — the timestamp and body are concatenated with a dot separator, hashed against the signing secret, and compared using `MessageDigest.isEqual` for constant-time comparison. A 5-minute replay window rejects stale timestamps.

Two event types come through: `sms.received` and `voice.received`. The voice events include a transcription field — Numbrrs transcribes incoming calls on their side — so a voice call arrives as an `InboundMessage` with the transcription as content and the call metadata (duration, recording URL) in the metadata map. If the transcription is empty, the content falls back to a descriptive string identifying the caller.

## Why this matters

The connector library now covers four mobile providers: `mobile-ref` for testing, `mobile-dial` and `mobile-twilio` for production telephony, and `mobile-numbrrs` for something none of the others can do — holding real registered identities on platforms that reject programmatic numbers. When the platform needs to test cross-platform messaging between Signal and Telegram, or verify that a WhatsApp integration works end-to-end, these are the numbers that make it possible.

It also validates something about the MobilePlatform SPI design. A provider that supports two out of six capabilities, with one of those being read-only, slots in without any special handling. The builder composition pattern and the `supports()` introspection were designed for exactly this kind of asymmetry — and seeing a genuinely minimal provider use them without friction confirms that the abstraction holds.

