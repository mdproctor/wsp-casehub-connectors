# Handoff — connectors

## Session 2026-11-26

### Completed this session

**#363** — `@QuarkusTest` integration test for manifest-driven synthetic bean wiring
**#365** — Manifest `requires`/`provides` declarations + `DependencyResolver` with sub-capability matching
**#366** — `RequireCapabilityPlugin` step type + `DependencyResolver` as CDI bean
**#367** — `provides` declarations for Telegram and Signal manifests
**#368** — Email as bootstrap primitive — resolver tests for dual SMS+email chain

### In progress

**#369 — Secure credential input via MCP browser** (branch: `issue-369-secure-credential-input`)

**Done:** Schema changes — `help` and `example` fields added to `FieldDeclaration`. All three manifests updated with help text, examples, and guidance on where to find credentials.

**Remaining:** The `credential-input` MCP tool that opens an MCP browser (Playwright or any available) with a single-page form. The form is auto-generated from manifest `auth.fields` — labels, help text, examples, password masking for sensitive fields. Credential values never touch the LLM context. Returns `{configured: true}` to the model.

### Queue

| # | Title | Status |
|---|-------|--------|
| connectors#369 | Secure credential input | In progress (schema done, MCP tool remaining) |
| connectors#370 | End-to-end setup chain test | Queued (depends on #369) |

### Cross-repo — platform

| # | Title | Status |
|---|-------|--------|
| platform#986 | Playbook runner handles unmet-requirement | Landed |
| platform#987 | ManualOption flow | Landed |

### Key design decisions (from this session)

- **Connectors never reference each other** — Signal declares `requires: sms[receive]`, not `requires: numbrrs`
- **Sub-capabilities** narrow along any dimension — `sms[receive]`, not `sms-receive`. Unqualified = full capability
- **ManualOption** — user satisfies requirement themselves (not a virtual connector)
- **Credential security** — API keys entered via browser form, never in chat. MCP tool returns success/failure only
- **Bootstrap primitives** — SMS + email are the foundation from which all other services chain

## References

- Spec: `specs/issue-365-manifest-requires/2026-11-25-manifest-requires-design.md`
- Blog: `blog/2026-11-26-mdp01-what-do-you-need.md`
