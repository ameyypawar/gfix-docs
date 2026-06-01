# Changelog

All notable changes to `gfix`. Versioning: `MAJOR.MINOR.PATCH-prerelease`. Alpha releases use `-alpha`, `-alpha.2`, etc.

## v0.1.0-alpha.6

Standalone CLI dispatch. All seven subcommands wired and shipped: `gfix merge`, `gfix preview`, `gfix resolve`, `gfix status`, `gfix log`, `gfix abort`, `gfix init`. Verified end-to-end against a real repo; `npm install -g @gitfix/cli@alpha` and `brew install ameyypawar/gfix/gfix` both resolve to alpha.6.

Release artifacts: https://github.com/ameyypawar/gfix/releases/tag/v0.1.0-alpha.6

## v0.1.0-alpha.3 (2026-05-26)

Hardening release. Two PRs landed: AI provider robustness (#101) and env-clear subprocess hardening (#102). Test count 120 -> 122.

**AI hardening (PR #101).** Closes #35, #42, #46.

- #35: `response_format: json_object` was unconditionally set in OpenAI requests. The o1/o3 reasoning model families reject it with HTTP 400. Now omitted for non-GPT-4/GPT-3.5 models, consistent with how Anthropic and Ollama callers already behave.
- #42: `pick_provider` had 70 lines of copy-paste per provider arm. Collapsed into a `KEY_PROVIDERS` lookup table — same precedence logic, half the code.
- #46: `ConflictGetResponse.ai_suggestion` was typed as `serde_json::Value`, causing the absence-of-suggestion case to serialize as `null` or `{}` rather than a typed `None`. Retyped as `Option<AiSuggestion>`; `null` now serializes correctly and is handled consistently by MCP clients.

**env_clear subprocess hardening (PR #102).** Closes #70 (P0), #72, #97, #99.

- Every git and subprocess invocation now routes through `gitfix_command()` — `env_clear()` plus an 8-key allowlist (`PATH`, `HOME`, `LANG`, `LC_ALL`, `TZ`, `TMPDIR`, `USER`, `LOGNAME`). Removes the last four parent-env leak sites identified after the #48 P0 fix.
- New regression tests: `finalize_does_not_inherit_hostile_git_dir`, `lookup_does_not_inherit_hostile_git_dir`.

**Tests.** 120 -> 122 passing.

**Install / upgrade.**

    brew upgrade gfix

Release artifacts: https://github.com/ameyypawar/gfix/releases/tag/v0.1.0-alpha.3

## v0.1.0-alpha — release date TBD, ships when ready

First public alpha. Closed-source binary distribution. Free during alpha.

**Capabilities.**

- CLI binary `gfix` + MCP server (`gfix mcp` subcommand) over stdio.
- 6 MCP tools: `gitfix_merge_preview`, `gitfix_merge_status`, `gitfix_conflict_get`, `gitfix_conflict_resolve` (+ `_batch`), `gitfix_merge_apply`, `gitfix_merge_abort`.
- Deterministic merge floor: text merge, then Mergiraf invoked as a subprocess (Mergiraf is GPL-3.0; `gfix` never links it).
- AI ceiling via MCP sampling (when the host advertises `sampling/createMessage`) or BYO-key direct HTTP. Supported providers: OpenAI, Anthropic, Gemini, Ollama. Single provider per call; precedence Anthropic -> OpenAI -> Gemini -> Ollama unless `GITFIX_BYOK_PROVIDER` is set.
- Cross-team rerere replay via `refs/gitfix/rerere/<blake3-hash>` — content-addressed, shareable via plain `git push`.
- Pushable audit refs at `refs/gitfix/audit/<merge_id>` — synthetic commits whose tree contains the full final merge state as `audit.json`.
- Untrusted-content framing: conflict bodies wrapped + defanged before AI prompt concatenation. Red-team corpus regression-guards the wrapper-escape vector found and fixed during M2.

**Platforms.**

- macOS arm64 — verified by developer dogfood (two sessions, audit refs `m_2026-05-22T10-25-45Z_c2bc70` and `m_2026-05-23T20-09-27Z_cebcd8`).
- macOS x86_64, Linux x86_64, Linux arm64 — binaries shipped; unverified by developer dogfood.

**Known limitations (tracked, lands alpha.2 or later).**

- Standalone CLI dispatch (`gfix merge feat/a feat/b` from a shell) — shipped in v0.1.0-alpha.6. See changelog entry above.
- `gfix doctor` config diagnostics.
- macOS XDG path divergence: `keys.toml` is read from `~/Library/Application Support/dev.gitfix.gitfix/keys.toml` instead of `~/.config/gitfix/keys.toml`. Documented in [setup.md](setup.md); proper fix at alpha.2.
- macOS binary is unsigned. Apple Developer ID code signing deferred to v1.0. Workaround in [setup.md](setup.md).

**Setup story.** [setup.md](setup.md) walks the Gemini Flash + Claude Code happy path. The eight findings from the M2 W1 dogfood log are addressed — most in docs, one in code, one as a documented workaround.

**Read more.** [posts/2026-06-launch.md](posts/2026-06-launch.md) — the M2 retrospective with full evidence from the M2 W1 + W2 dogfood sessions, including the cross-machine rerere replay test (5-minute wall-clock, zero AI calls).
