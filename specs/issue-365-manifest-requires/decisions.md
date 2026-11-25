# Decisions — #365 Manifest Requires

## D1: Requires abstraction level

**Choice:** Capability-level — connectors declare abstract needs (e.g. "sms"), never name specific providers
**Alternatives:**
- Connector-level (requires: [{connector: numbrrs}]) — simpler but hardcodes resolution, violates separation
- Both capability + preferred provider — adds coupling without clear benefit
**Rationale:** One service should never mention another. Signal needs SMS; it doesn't know or care about Numbrrs. Resolution is orthogonal.
**Trade-offs:** The lifecycle system must maintain a capability vocabulary; typos in capability names won't be caught at parse time
**Sources:** Signal registration playbook (hardcodes connector-status: numbrrs — the anti-pattern), user direction
**Exploration:** quick
**Status:** captured

## D2: Provider capability declaration

**Choice:** Providers self-declare capabilities via explicit `provides` list in manifest
**Alternatives:**
- Derive from platform + produces — less explicit, harder to reason about which subset of platform capabilities a connector offers
**Rationale:** A connector may provide only a subset of its platform's capabilities (Numbrrs provides sms but not voice-call). Explicit declaration is unambiguous.
**Trade-offs:** Another field to maintain in manifests; could drift from actual implementation
**Sources:** Numbrrs manifest (platform: mobile but only provides Sms + NumberInfo), ConnectorManifest.java
**Exploration:** quick
**Status:** captured

## D3: User-provided resolution model

**Choice:** Manual resolution step — the lifecycle system recognises some capabilities can be satisfied by human action
**Alternatives:**
- Virtual user-provided connector — uniform model but over-engineered for "ask the human"
**Rationale:** When a user has their own phone number, there's no connector to set up. Manual resolution is a strategy the resolver offers, not a connector bean.
**Trade-offs:** Manual resolution needs its own codepath in the resolver; not everything is a connector
**Sources:** User direction
**Exploration:** quick
**Status:** captured

## D4: Resolver architecture

**Choice:** Static dependency graph built at startup from discovered manifests
**Alternatives:**
- Capability registry with dynamic lookup — adds runtime state, over-engineered for 1-2 chain depths
- Playbook-only resolution — no manifest changes but dependencies not discoverable without parsing playbooks
**Rationale:** Declarative, manifest is already the source of truth, resolver is a pure function over the graph. No runtime state beyond what's already loaded.
**Trade-offs:** Can't discover dependencies that aren't in manifests (but that's the point — make them explicit)
**Sources:** ConnectorManifestLoader.java, ConnectorLifecycleProcessor.java, signal-registration.yaml
**Exploration:** quick
**Status:** captured

## D5: Capability granularity

**Choice:** Capabilities are a category with optional sub-capabilities — `sms` is the category, `[receive]` narrows it
**Alternatives:**
- Directional suffixes (sms-receive, sms-send) — privileges one dimension, vocabulary explosion risk
- Flat specific names (inbound-sms, outbound-sms) — loses grouping, no structure
**Rationale:** Send/receive is just one dimension of capability. Sub-capabilities allow narrowing along any dimension without baking a taxonomy into the naming convention. Unqualified `sms` means full capability; `sms[receive]` narrows to a subset.
**Trade-offs:** Matching logic is slightly more complex (must check sub-capability inclusion); unqualified providers match all qualified requirements
**Depends on:** D1 (capability-level abstraction), D2 (provider self-declaration)
**Sources:** Numbrrs (receives but can't send SMS), user direction
**Exploration:** quick
**Status:** captured
