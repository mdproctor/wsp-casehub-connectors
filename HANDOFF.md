# HANDOFF — casehub-connectors

## Last Session

Landed 4 issues (#289–#292) plus a partial fix for #288. Filed 2 new issues (#293, #294). All work landed on main and published.

**#289+#290 — MicrosoftGraphClient URI handling:**
- Replaced deprecated `java.net.URL(String)` in `toUri()` with manual URI component parsing (scheme, authority, path, query, fragment)
- Unified all 5 `BASE_URL + path` call sites through `toUri()` — `putBytes`, `getBytes`, `getRaw` were bypassing it via `URI.create()`
- `getBytesFromUrl` and pagination cursors correctly stay on `URI.create()` (pre-formed URLs)

**#291 — SpeakerPlugin configurable sample rates:**
- Added `@Optional List<Integer> supportedSampleRates` to the plugin record
- Default `Set.of(16000)` preserves backward compatibility
- Error message now lists configured valid rates on rejection

**#292 — Cloned-voice attack playbook:**
- Rewrote 06b negative twin to demonstrate actual voice cloning attack
- Bob registers with `bm_daniel`, attacker uses same voice, passes speaker ID gate
- Test asserts the clone passes — proving the vulnerability multi-factor prevents

**#288 (partial) — Playbook integration tests:**
- Fixed `mobile-send-sms` params in 06-joint-account-confirmation (`platform: ref`, `body` instead of `message`)
- Root cause identified: platform YAML runtime 0.4-20261118 SNAPSHOT broke `capture:` variable scoping — all 18 tests fail with "Unknown prefix"
- Blocked on platform fix (filed as #294)

**Filed:**
- #293 — `toUri()` treats `#` and `?` as URI delimiters in file paths (known limitation, not a regression)
- #294 — Platform YAML runtime capture variable scoping broken (blocks #288)

**Blog:** "Deprecation, Encoding, and Cloned Voices" — URI migration, sample rates, voice cloning attack demo, platform regression discovery.

## Queue

13 items remaining. Next up: #288 (blocked on #294), then #264 (GeneratePlugin annotations).

## Key Context for Next Session

- The platform YAML runtime regression (#294) blocks all playbook integration tests. The Nov 16 SNAPSHOT (279KB) works; the Nov 18 SNAPSHOT (331KB) doesn't. `capture:` variable scoping is broken — step results are no longer accessible as named prefixes.
- #288 has the `mobile-send-sms` parameter fix committed and landed, but the broader test failures need the platform fix first.
- MicrosoftGraphClient is now fully consistent — every `BASE_URL + path` goes through `toUri()`. Only pre-formed URLs (CDN, cursors) use `URI.create()` directly.
