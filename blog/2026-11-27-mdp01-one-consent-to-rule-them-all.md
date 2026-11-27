---
layout: post
title: "One Consent to Rule Them All"
date: 2026-11-27
entry_type: note
subtype: diary
projects: [casehubio/connectors]
tags: [architecture, connectors, lifecycle, oauth, family-model]
series: issue-371-oauth-families-persist-cred
---

# One Consent to Rule Them All

The connector lifecycle now has manifests and dependency chains. But five Google modules and three Microsoft modules still sit outside it — each with its own `*Beans.java` CDI producer, each wiring `ServiceConnectionProvider` by hand. They share a client ID and client secret, yet nothing in the system knows they're related.

The family model fixes this. A family manifest declares what a group of connectors share: OAuth endpoints, scopes, client credentials, and consent strategy. Each member manifest references the family by name and inherits everything. After resolution, the manifest is self-contained — no downstream code needs to know families exist.

## The Design Tension

The interesting constraint here isn't the YAML. It's the wiring gap between the Phase 1 synthetic bean pipeline and what OAuth connectors actually need.

Phase 1 connectors take credential strings as constructor parameters — `NumbrrsMobilePlatform(String apiKey, String numbers)`. The build extension resolves `field: api-key` via `ProviderCredentialFacade` and passes the value. Clean, uniform, works.

OAuth connectors don't take credential strings. They take `ServiceConnectionProvider` — a CDI singleton that handles token retrieval, refresh, and scope management. The provider name ("google", "microsoft") is a call-time parameter, not a bean discriminator. Every Google connector, every Microsoft connector, and GitHub all inject the same singleton instance.

I expected this to need a new constructor-arg type — `service-connection: google` to tell the pipeline "inject ServiceConnectionProvider scoped to this provider." Claude's decision review caught it immediately: `ServiceConnectionProvider` is already a CDI bean. The existing `bean:` arg type handles it. No new mechanism needed.

That single insight collapsed three days of planned work into nothing. The manifest for a Google connector looks like this:

```yaml
connector:
  id: calendar-google
  platform: calendar
  auth:
    type: oauth2
    family: google
  produces:
    - class: ...GoogleCalendarPlatform
      type: platform
      constructor-args:
        - bean: ...ServiceConnectionProvider
```

No new syntax. No special handling. The existing pipeline already knows how to inject beans.

## Family Resolution

![Family resolution pipeline](images/family-resolution-pipeline.svg)

The resolution happens inside `ConnectorManifestLoader.discoverFromClasspath()`. After scanning `META-INF/connector-manifest.yaml` files, it calls `FamilyManifestLoader.discover()` to find `META-INF/families/*.yaml`, then runs `enrichWithFamily()` on every manifest whose `auth.family` is set. The enrichment replaces the member's empty `auth.fields` with the family's field definitions and populates `auth.oauth` with the family's OAuth config.

After this, every consumer sees the same resolved data. The build extension, the registry, the MCP tools — none of them know about families. They just see a manifest with fields, OAuth config, and a `credentialKey()` that returns "google" instead of "calendar-google". Shared credentials, one consent flow, zero family-specific branching downstream.

## Two Consent Strategies

Google supports incremental consent natively — request Calendar scopes now, Drive scopes later, no re-prompt for what's already granted. Microsoft doesn't — all scopes must be declared upfront. Making this a universal policy would have been wrong for at least one provider.

The family manifest declares its own strategy: `consent: incremental` for Google, `consent: aggregated` for Microsoft. The consent flow reads it and behaves accordingly. The field is there for the future — Phase 3 always requests all declared scopes, because true per-module activation needs a trigger that doesn't exist yet. But the data model is ready.

## What's Built

Three batches landed: the `FamilyManifest` record and loader, the build extension changes (the `value:` constructor-arg kind, `@Named` qualifier support, `OAuthConfigRegistry`), and `ConnectorRegistry.updateStatus()` for runtime re-evaluation when credentials change. The consent flow, module migrations, and end-to-end tests are next.

The Microsoft migration has one interesting wrinkle. `DocumentMicrosoftBeans` produces two `MicrosoftDrivePlatform` instances — one for OneDrive (`/me/drive`), one for SharePoint (`/sites/{siteId}/drive`). The `qualifier` field on `BeanDeclaration` and the `value:` constructor-arg exist specifically for this case: same class, different configuration, disambiguated by `@Named`.
