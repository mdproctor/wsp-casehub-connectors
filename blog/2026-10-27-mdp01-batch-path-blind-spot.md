---
layout: post
title: "The Batch Path's Blind Spot"
date: 2026-10-27
entry_type: note
subtype: diary
projects: [casehubio/connectors]
tags: [graphql, document-platform, playbooks]
---

# The Batch Path's Blind Spot

Ten platform SPIs. Ninety-two GraphQL operations across nine of them. And one connector — documents — stuck at a single `syncDocuments` call while its SPI exposed four full capability interfaces.

The gap wasn't an oversight in the usual sense. When `DocumentPlatform` was built, sync was the only operation the platform needed at the GraphQL layer. Individual file operations went through MCP tool calls and that worked fine. But the playbook engine in `life` changed the equation. Playbooks batch connector operations over GraphQL *before* LLM involvement — mechanical pre-staging and post-staging that avoids burning tokens on operations a script could handle. A playbook that needs to store a report, search for an existing document, or generate a share link couldn't use the batch path. It would fall back to individual MCP calls, which defeats the entire purpose of mechanical coordination.

The fix was straightforward — `ConnectorContactsApi` already had the pattern: capability-checked methods with `requireCapability`, `@PlatformQuery`/`@PlatformMutation` annotations, `UnsupportedCapabilityException` with recovery metadata. We added ten operations covering all four `DocumentPlatform` interfaces:

- **FileOperations** — `listFiles`, `getFile`, `downloadFile`, `uploadFile`, `deleteFile` (plus the existing `syncDocuments`)
- **FolderOperations** — `listFolders`, `createFolder`, `moveFile`
- **SearchOperations** — `searchDocuments`
- **SharingOperations** — `getShareLink`

Each operation checks `platform.supports()` before calling the capability sub-interface. If a provider doesn't support folders (say, a flat-file backend), the caller gets a structured `UnsupportedCapabilityException` listing what *is* available — the same self-correction metadata pattern that lets LLMs recover gracefully from capability mismatches.

What makes this worth noting beyond the code: it closes the last significant gap between the MCP direct-call path and the GraphQL batch path. Every platform SPI now has enough GraphQL surface for playbooks to stage their connector work mechanically. The four `life` workers that reference `sync_documents` — `produce-report`, `care-plan`, `confirmation`, `record-completion` — can now search for existing files, upload generated content, and share results without dropping out of batch mode.
