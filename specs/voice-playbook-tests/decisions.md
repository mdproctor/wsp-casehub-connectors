# Decisions — Voice Playbook Integration Tests

## D1: Test execution mechanism

**Choice:** ConnectorTestEngine extending YamlTestEngine
**Alternatives:**
- Java AgentExampleExecutionTest pattern — more control but Java boilerplate, not clean YAML showcase
- Java bootstrap + YAML tests hybrid — unnecessary complexity when YamlTestEngine already handles discovery
**Rationale:** YamlTestEngine is the platform's newest testing infrastructure. Extending it lets us pre-register connector platform SPIs into the service registry while keeping test definitions pure YAML.
**Trade-offs:** Coupled to YamlTestEngine's API surface; requires understanding Walker resolution
**Sources:** YamlTestEngine.java, AgentExampleExecutionTest.java, SimulationRuntime.java
**Exploration:** quick
**Status:** captured

## D2: Testing modes (simulation vs real LLM)

**Choice:** Profile-gated — same .test.yaml files, Maven profile swaps backend
**Alternatives:**
- Separate test files — duplicates test structure
- Manifest override in YAML — requires platform-side changes
**Rationale:** Same playbooks validate in both modes. Default runs with simulate: block (offline, fast). -Preal-llm activates manifest-based LLM backend. Platform APIs stay simulated in both modes.
**Trade-offs:** Real LLM tests require credentials and are non-deterministic
**Sources:** Platform CorpusBackend, AgentSessionTestSupport
**Exploration:** quick
**Status:** captured

## D3: YAML step syntax for platform APIs

**Choice:** Domain-operation combined-key steps — one plugin per connector domain
**Alternatives:**
- Individual @Plugin per operation — O(n) Java classes as SPIs grow
- Generic platform step with nested domain/operation — verbose, not idiomatic
**Rationale:** Walker's combined-key parsing splits `bank balance: "acc-123"` into plugin="bank", primaryInput="balance", _value="acc-123". One BankPlugin dispatches to the right SPI capability method. New SPI methods surface without new Java classes.
**Trade-offs:** Domain plugin dispatcher grows as operations are added; all operations share one plugin class
**Sources:** Walker.java lines 280-305 (combined-key resolution), ConnectorBankApi.java (GraphQL naming)
**Exploration:** deep-analysis
**Status:** captured

## D4: Module structure

**Choice:** connector-playbook-tests module with src/main (infrastructure + playbooks) and src/test (test YAMLs)
**Alternatives:**
- Top-level examples/ directory — not Maven convention
- Split into connector-test-engine + integration-tests — over-engineering for current scope
**Rationale:** Follows platform's playbooks/ module pattern. Domain plugins and ConnectorTestEngine in src/main are publishable as reusable infrastructure. Playbook YAMLs in src/main/resources are publishable examples. Test YAMLs in src/test.
**Trade-offs:** New module adds to build; could have been in integration-tests
**Sources:** Platform playbooks/src/main/playbooks/agent-interaction-patterns/
**Exploration:** quick
**Status:** captured

## D5: Platform API coverage

**Choice:** Voice + Bank + WhatsApp (chat) + Email — bi-directional pipelines with simulation
**Alternatives:**
- Voice-only — too narrow to showcase multi-platform
- All 11 platforms — too broad for initial showcase
**Rationale:** These four platforms demonstrate distinct interaction patterns: voice (streaming audio), banking (query/response), chat (bi-directional messaging), email (async notification). All have reference implementations available.
**Trade-offs:** Travel and health deferred to follow-up
**Sources:** ReferenceVoicePlatform, ReferenceBankPlatform, ReferenceChatPlatform, ReferenceEmailPlatform
**Exploration:** quick
**Status:** captured
