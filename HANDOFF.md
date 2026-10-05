# Handover — casehub-connectors

## What Happened

Completed the ref/simulation unification (#138 design, #141 Phase 1, platform#512 Phase 2a, #143 Phase 2b). All 9 ref implementations normalised, seed data extracted to YAML, simulation corpus files shipped for 6 interpretive capability areas. DataRealism E2E verification landed in platform#517.

## Current Branch: issue-139-simulation-integration-audit

**Status:** Scaffolded, no implementation yet.

**Issue #139:** Audit each connector SPI's ref implementation to determine whether it would benefit from simulation framework integration. Categorise every capability as seed-and-go (ref logic is sufficient) or seed-plus-simulate (needs simulation-driven responses).

**Context:** The #138 design spec already contains the interpretive capability classification (§ Interpretive Capability Classification). The seed data YAML and simulation corpus files from #143 already implement this classification. This audit is a verification pass — confirm the classification holds, identify any gaps in corpus coverage, and document the assessment formally.

**The issue asks for 4 deliverables:**
1. Per-SPI assessment table (seed-and-go vs seed-plus-simulate per capability)
2. Priority ordering — which SPIs benefit most from simulation integration
3. Common patterns that can be extracted (search simulation, CRUD ref)
4. Impact on demo scenarios — which SPIs are hard to demo without real providers

**What's already done (from #138 and #143):**
- Interpretive capability classification exists in the design spec
- Simulation corpus files shipped for all 6 interpretive areas (commerce, location, contacts, document, email, project)
- Seed data YAML files for all 7 ref modules with data

**What remains:**
- Formal assessment document with the per-SPI table
- Verify corpus coverage matches the classification (are there gaps?)
- Priority ordering and demo impact assessment
- Close the issue

## Decisions

- `DataRealism` enum values: `GARBAGE, PLACEHOLDER, STRUCTURALLY_VALID, DOMAIN_PLAUSIBLE, RECORDED_REAL` (in `simulation-api`)
- Interpretive capabilities get `STRUCTURALLY_VALID` from ref fallthrough, strategy-level realism from simulation
- Deterministic capabilities stay unmarked (authoritative)

## Key Artifacts

- Design spec: `docs/specs/issue-138-design-unify-ref-simulation/2026-10-04-unify-ref-simulation-design.md`
- Decisions: `docs/specs/issue-138-design-unify-ref-simulation/decisions.md`
- Contributor guide seed data section: `docs/guides/contributor-guide.md` (§ Ref Implementation Seed Data)
