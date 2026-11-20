---
layout: post
title: "Seed data, your data — making ref providers user-editable"
date: 2026-11-20
entry_type: note
subtype: diary
projects: [casehubio/connectors]
tags: [architecture, mcp, ref-providers, seed-data, design]
series: issue-329-seed-externalisation
---

# Seed data, your data — making ref providers user-editable

The standalone MCP server ships 13 ref providers with realistic seed data — calendar events, bank transactions, medical records, product catalogues. Connect an LLM agent, and you've got 133 tools backed by working data in under a second. The problem: that data is baked into the production JARs. Every consumer of a ref module carries seed YAML they'll never use. And standalone MCP server users who want to model their own domain — a dental practice instead of a GP, a hardware shop instead of an electronics catalogue — have to rebuild the project to change it.

The fix is a three-tier resolution pattern that separates where data lives from how it's loaded.

## The loading pattern before

Every ref module had a `SeedLoader` class — 12 of them, all copy-pasted, all doing the same thing:

```java
static List<CalendarInfo> loadCalendars() {
    try (var is = SeedLoader.class.getResourceAsStream("/seed/calendars.yaml")) {
        if (is == null) throw new IllegalStateException("Missing /seed/calendars.yaml");
        return YAML.readValue(is, new TypeReference<>() {});
    }
    // ...
}
```

Classpath only. No fallback. Missing file = crash. The YAML sat in `src/main/resources/seed/` and shipped in the production JAR whether you wanted it or not.

## Three-tier resolution

The replacement is a shared `SeedDataResolver` in `connectors-api`:

```java
public static Optional<InputStream> resolve(String domain, String filename) {
    Path dir = seedDir;
    if (dir != null) {
        Path file = dir.resolve(domain).resolve(filename);
        if (Files.exists(file))
            return Optional.of(Files.newInputStream(file));
    }
    InputStream is = SeedDataResolver.class
        .getResourceAsStream("/seed/" + domain + "/" + filename);
    return Optional.ofNullable(is);
}
```

Three tiers: file-based directory first (user-provided corpus), classpath second (test resources), empty third (graceful degradation — the provider starts with no data, no crash). The `domain` parameter (`"calendar"`, `"bank"`, `"mobile"`) namespaces everything so there's no collision on the classpath or on disk.

Each SeedLoader becomes a thin wrapper:

```java
var maybeIs = SeedDataResolver.resolve("calendar", "calendars.yaml");
if (maybeIs.isEmpty()) return List.of();
try (var is = maybeIs.get()) {
    return YAML.readValue(is, new TypeReference<>() {});
}
```

The `connectors-api` module stays zero-dependency — `SeedDataResolver` uses only `java.nio.file` and `java.io`. A CDI startup observer in the MCP server module reads the Quarkus config property and calls `setSeedDir()`:

```java
@ConfigProperty(name = "casehub.ref.seed-dir")
Optional<String> seedDir;

void onStart(@Observes StartupEvent ev) {
    seedDir.map(Path::of).ifPresent(SeedDataResolver::setSeedDir);
}
```

## The collision that code review caught

I didn't see this during implementation. Claude caught it in the review pass: both `email-ref` and `mobile-ref` have a file called `messages.yaml`. Before the fix, the classpath fallback was `/seed/messages.yaml` — domain-unaware. When both modules are on the classpath (as they are in the MCP server, in playbook tests, and in integration tests), `getResourceAsStream` returns whichever the classloader finds first. The mobile SeedLoader silently loads email data. Jackson either deserialises it into the wrong type and you get garbage, or it throws a parse error with no indication that the wrong file was loaded.

The fix was straightforward: include the domain in the classpath path — `/seed/calendar/calendars.yaml` instead of `/seed/calendars.yaml`. Both the file-based and classpath tiers now use the same `<domain>/<filename>` structure. But it's the kind of bug that's invisible until you have enough modules on the classpath at the same time, and it would have been genuinely confusing to debug in the field.

## What the MCP server user sees

The standalone distribution now looks like this:

```
quarkus-app/
  quarkus-run.jar
  corpus/
    bank/accounts.yaml, balances.yaml, transactions.yaml
    calendar/calendars.yaml, events.yaml
    chat/channels.yaml, members.yaml
    commerce/products.yaml
    contacts/contacts.yaml, groups.yaml
    document/files.yaml, folders.yaml
    email/messages.yaml
    health/appointments.yaml, providers.yaml, records.yaml
    location/geocoding.yaml, places.yaml
    mobile/conversations.yaml, messages.yaml, numbers.yaml, voicemails.yaml
    project/project-data.yaml
    travel/accommodation.yaml, transport.yaml
```

Edit a YAML file, restart the server, test with your LLM agent. No rebuild. No code changes. Want a veterinary practice instead of a GP? Edit `health/providers.yaml`. Want French restaurants instead of English ones? Edit `location/places.yaml`. The ref providers are now a prototyping tool, not just a testing scaffold.

For consumers who use real providers (Google Calendar, Slack, TrueLayer), the production JARs are smaller — no seed YAML shipped. The ref providers start empty and serve as pure in-memory backends for whatever data the application injects at runtime.

The next step is a first-run bootstrap: detect that the corpus directory is missing and prompt the user to extract defaults. That's a separate piece of work — the infrastructure is in place, the UX is not yet.
