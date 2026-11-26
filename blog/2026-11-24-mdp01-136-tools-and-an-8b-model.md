---
layout: post
title: "136 tools and an 8B model"
date: 2026-11-24
entry_type: note
subtype: diary
projects: [casehubio/connectors]
tags: [langchain4j, mcp, llm, integration-testing, voice]
---

# 136 tools and an 8B model

The MCP server exposes 136 tools across 13 platform domains. That's useful for a human navigating with a tool list, but what happens when you hand all 136 to an LLM and ask it to pick the right one?

I wanted to find out with real hardware — no cloud API keys, no rate limits, just a local Ollama instance running llama3.1:8b. The question wasn't whether a frontier model could handle it. The question was whether the tool schemas were clean enough for a small model to get it right most of the time.

## The schema problem

They weren't. LangChain4j 1.0.0-beta5 chokes on JSON schemas that use `$ref` and `$defs`. The platform's `DynamicToolRegistrar` builds schemas via `addArgument()`, which produces nested `$ref` pointers for record types. LangChain4j's schema parser expects flat, self-contained schemas — it silently drops any tool whose schema contains a reference.

We built `InlineSchemaBuilder` in platform to produce self-contained schemas: records get inlined as nested `properties` objects, sealed interfaces become `oneOf` arrays, enums become `enum` string arrays. No `$ref`, no `$defs`, no external references. The builder handles `BigDecimal`, `LocalDate`, `LocalTime`, `Instant`, `UUID`, and recursive record hierarchies.

The fix landed in `DynamicToolRegistrar.registerOperationTool()` — one call replacement, `setInputSchema()` with the inlined schema instead of `addArgument()` calls.

## What passed, what didn't

Eight direct tool tests — no LLM, just verifying the MCP server returns correct results for explicit tool calls — pass at 100%. These cover voice (list voices, speak, transcribe), commerce (search, cart, checkout), health (search providers, book appointment), and cross-domain scenarios.

Fifteen LLM-driven tests let the model choose which tool to call, then verify the result. Each run lands 11–12 out of 15. The 3–4 failures rotate: sometimes the model picks `search_contacts` instead of `get_contact`, sometimes it calls `search_products` with the wrong parameter structure. The failures are non-deterministic and inherent to running an 8B parameter model against 136 candidate tools.

That's genuinely useful signal. A frontier model would get all 15. The 8B model's mistakes tell us which tool descriptions are ambiguous — `search_contacts` and `get_contact` have overlapping natural-language descriptions, and the model reasonably picks either one.

## Voice on the wire

The MCP server now has real speech. Voice-sherpa runs Kokoro (TTS) and Whisper (STT) locally, and the server wires them in through the existing `VoicePlatform` SPI. A `voice-config.yaml` in the server resources declares the available voices; `VoiceConfigProducer` reads it at startup and populates the registry.

The round-trip works: ask the model to speak something, get audio bytes back, feed them to transcription, verify the text matches. The direct test for this passes reliably because there's no tool-selection ambiguity — the model is told exactly what to do.

## The flaky line

Twelve out of fifteen is the wrong number if you want a green CI pipeline. It's the right number if you want to know whether your tool schemas are machine-readable. We marked the LLM tests with a `local-llm` profile — they run when you have Ollama up, they're skipped in CI. The direct tests run everywhere.
