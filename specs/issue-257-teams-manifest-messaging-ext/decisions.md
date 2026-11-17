## D1: Deliverable format — standalone doc + manifest template

**Choice:** Two artifacts: a standalone setup guide (`docs/guides/teams-messaging-extensions-setup.md`) and an in-repo manifest template (`chat-teams/src/main/resources/manifest-template.json`)
**Alternatives:**
- Consumer guide section only — not enough room for the full setup journey, crowds existing content
- Standalone doc without template — forces consumers to construct manifest JSON from scratch
- Template only — missing the bot registration and commandId routing context
**Rationale:** The template is the actionable artifact consumers copy and customise. The standalone doc provides the narrative: prerequisites, manifest walkthrough, handler wiring, and commandId routing. Neither is complete without the other.
**Trade-offs:** Two files to maintain instead of one — acceptable given they serve different purposes (reference vs. starting point)
**Sources:** Issue #257 body, docs/guides/consumer-guide.md (existing structure)
**Exploration:** quick
**Status:** captured

## D2: Manifest template scope — all three extension types

**Choice:** Single template with search command, action command, and link unfurling (`messageHandlers`) entries, with comments explaining each section
**Alternatives:**
- Minimal (search only) — forces consumers to construct action/unfurl entries from docs
- Per-type snippets in the doc — no single copyable starting point
**Rationale:** All three types are compact in the manifest. Consumers delete what they don't need, which is easier than constructing from snippets. The existing SPI supports all three, so the template should demonstrate all three.
**Trade-offs:** Template is slightly larger — but `composeExtensions` entries are small
**Sources:** Bot Framework manifest schema, InvokeHandler SPI (#236)
**Exploration:** quick
**Status:** captured

## D3: Azure bot registration coverage — checklist with links

**Choice:** Bullet checklist of what must be configured, linking to Microsoft's official docs for procedural detail
**Alternatives:**
- Full step-by-step walkthrough — rots as Microsoft changes their portal UI
- Prerequisites only — too terse, consumers won't know what to configure
**Rationale:** Microsoft's portal UI changes frequently. A checklist gives consumers enough to know what to do without us maintaining how to click through Azure. The doc's real value is the manifest structure and InvokeHandler wiring.
**Trade-offs:** Consumers need Microsoft docs open alongside ours — acceptable since the Azure steps are standard
**Sources:** Microsoft Bot Framework registration documentation
**Exploration:** quick
**Status:** captured

## D4: commandId routing documentation — both levels

**Choice:** Document both the activity name dispatch (`InvokeHandler.name()` → `composeExtension/query` etc.) and the `commandId` routing within handlers, with a worked example
**Alternatives:**
- Activity name only — leaves consumers stuck when they deploy multiple search commands and can't distinguish them
**Rationale:** The `commandId` routing is the non-obvious part. A consumer deploying a real extension with multiple search commands immediately hits "how do I distinguish command A from command B inside one handler?" — the doc should answer this.
**Trade-offs:** None significant — the worked example is small
**Sources:** Bot Framework composeExtension invoke activity schema, InvokeRequest.java (value field carries commandId)
**Exploration:** quick
**Status:** captured
