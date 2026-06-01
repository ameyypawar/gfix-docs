# gfix setup

End-to-end install + first-merge walkthrough. ~15 minutes start to first audit ref.

Path through this doc:
1. [Install](#1-install)
2. [macOS unsigned-binary workaround](#2-macos-unsigned-binary-workaround) — **read this first if you're on macOS**
3. [Wire gfix into Claude Code (MCP)](#3-wire-gfix-into-claude-code-mcp)
4. [Configure an AI provider (BYO-key)](#4-configure-an-ai-provider-byo-key)
5. [Verify the install](#5-verify-the-install)
6. [Troubleshooting](#6-troubleshooting)
7. [Where to get help](#7-where-to-get-help)

---

## 1. Install

Three install paths. Pick whichever fits your setup. All three land the same binary.

### macOS (Homebrew, recommended)

```sh
brew install ameyypawar/gfix/gfix
```

Homebrew installs to `/opt/homebrew/bin/gfix` on Apple Silicon, `/usr/local/bin/gfix` on Intel macOS. The rest of this doc uses `/opt/homebrew/bin/gfix` — substitute your actual path.

### Linux + macOS (curl)

```sh
curl -fsSL https://gfix.space/install | sh
```

Installs to `/usr/local/bin/gfix` by default. Override with `PREFIX=/some/other/path`. The script verifies SHA256 against the published manifest before installing; mismatch exits non-zero.

### Any Node-equipped machine (npm)

```sh
npm install -g @gitfix/cli@alpha
```

Installs the platform-appropriate binary via the npm `optionalDependencies` pattern. The `@alpha` tag is required — the `latest` tag currently points at alpha.4; `@alpha` resolves to alpha.6 (the version with the full CLI surface wired). Useful inside Docker images or CI containers that already have Node.

---

## 2. macOS unsigned-binary workaround

**Read this before running `gfix --version` on macOS.** The alpha binary is unsigned (Apple Developer ID code signing is on the v1.0 roadmap — $99/year, deferred until traction warrants). On first launch, macOS Gatekeeper shows:

> "gfix" can't be opened because it is from an unidentified developer.

The error dialog does NOT mention the fix. Clear the quarantine attribute:

```sh
xattr -d com.apple.quarantine /opt/homebrew/bin/gfix
```

(Use `/usr/local/bin/gfix` if that's where Homebrew or curl installed it. The Homebrew `caveats` block prints the right path after `brew install`.)

Then `gfix --version` works. The quarantine attribute is set once per download — you won't need this command again on the same binary.

**If `xattr -d com.apple.quarantine` fails** (rare; happens on encrypted volumes and some NFS mounts): open **System Settings -> Privacy & Security**, scroll to the "Security" section, find the "gfix was blocked" entry, click "Open Anyway", confirm.

**Why not just sign the binary.** Apple Developer ID is $99/year and a one-person indie product in alpha doesn't have a return on that yet. Signing lands at v1.0 when the product has either revenue or enough install volume to make the friction the wrong trade-off.

---

## 3. Wire gfix into Claude Code (MCP)

Register `gfix` as an MCP server in Claude Code. Two flags matter — get both right or it won't work:

```sh
claude mcp add --scope user -- gfix /opt/homebrew/bin/gfix mcp
```

Why each piece:

- **`--scope user`** — without this, the registration is project-scoped to the directory you ran `claude mcp add` in. Opening Claude Code in a different repo means gfix is invisible there. `--scope user` makes gfix available from any directory.
- **`--`** — the separator between flags and positional args. Without it, Claude CLI's `-e` flag (env vars; variadic) consumes the next positional as a second env value. Without `--`, you get `"missing required argument 'commandOrUrl'"`, which is cryptic. Use `--` even when you have no `-e` flags; it's a forward-compatible habit.
- **`gfix`** — the name the MCP host uses to refer to the server. Conventionally matches the binary name.
- **`/opt/homebrew/bin/gfix mcp`** — the actual command to launch the server. The `mcp` subcommand puts gfix into MCP-server-on-stdio mode.

With env vars (for BYO-key — see §4):

```sh
claude mcp add --scope user -e GITFIX_BYOK=1 -- gfix /opt/homebrew/bin/gfix mcp
```

Verify the registration:

```sh
claude mcp list
```

You should see `gfix` in the user-scope list. Re-open Claude Code (or run `/mcp` in any chat) to confirm the tools are loaded; you'll see `gitfix_merge_preview`, `gitfix_conflict_get`, `gitfix_conflict_resolve`, `gitfix_merge_apply`, `gitfix_merge_abort`, `gitfix_merge_status` (and `gitfix_conflict_resolve_batch`).

### Other MCP hosts

`gfix mcp` is a vanilla MCP-over-stdio server. Anything that speaks MCP should work — Cursor, Codex, Aider, opencode, Continue, Gemini CLI. The Claude Code recipe above translates directly: register `/opt/homebrew/bin/gfix mcp` as a stdio MCP server with the same env vars. Alpha.1 is dogfood-tested against Claude Code; other hosts are best-effort, file issues for friction.

---

## 4. Configure an AI provider (BYO-key)

gfix's AI ceiling has two paths:

- **MCP sampling** — when the host (Claude Code, Cursor, etc.) advertises `sampling/createMessage`, gfix asks the host's LLM. Your model, your bill, your privacy posture. **Today, Claude Code does NOT advertise sampling.** This will change; for now, BYO-key is the primary AI path.
- **BYO-key** — gfix calls the provider's HTTP API directly using a key you supply. Set `GITFIX_BYOK=1` in the MCP server env to enable.

### Which provider to pick

For first-time users in 2026: **Gemini Flash, free tier, no billing required.** Reasoning:

- **OpenAI** walls brand-new accounts immediately — no free API credits by default; first call 429s. Documented friction.
- **Anthropic** API requires billing setup; small free credit but not zero-billing.
- **Ollama** works offline but the macOS arm64 compile is multi-hour on first install; not a smooth onboarding.
- **Gemini Flash** has a working free tier with no billing setup. Recommended onboarding.

### keys.toml — the only sanctioned place to put keys

**Never paste an API key into a terminal command** (`-e API_KEY=sk-...`) or a chat window. The `-e` flag is documented for non-sensitive config only. Use `keys.toml`.

**Path matters and is platform-specific.** On macOS, the `directories` Rust crate resolves to the Apple convention path; on Linux it follows XDG.

| Platform | keys.toml path |
|---|---|
| macOS | `~/Library/Application Support/dev.gitfix.gitfix/keys.toml` |
| Linux | `~/.config/gitfix/keys.toml` |

If you're not sure which path gfix is reading, `gfix --version` prints the resolved path. Edit that one.

(Yes, the macOS path is non-obvious and diverges from XDG. We know. Proper fix — honor `XDG_CONFIG_HOME` on all platforms — lands at alpha.2. For alpha.1, the workaround is to write to the path `gfix --version` prints.)

### Minimal keys.toml for Gemini Flash

Create the file (substitute the macOS path if you're on macOS):

```sh
mkdir -p ~/.config/gitfix
$EDITOR ~/.config/gitfix/keys.toml
```

Contents — **the `[providers.gemini]` section header is required**. Without it, gfix parses the file successfully but reads the provider block as empty, with no diagnostic. You'll get silent failures on the AI path until the header is in place:

```toml
[providers.gemini]
api_key = "AIzaSy...your-key-here..."
model = "gemini-2.0-flash"
```

Get a Gemini API key at [aistudio.google.com/apikey](https://aistudio.google.com/apikey). Free tier; no billing required.

### Other providers (Anthropic, OpenAI, Ollama)

Same shape — one `[providers.<name>]` block per provider:

```toml
[providers.anthropic]
api_key = "sk-ant-..."
model = "claude-3-7-sonnet-20250219"

[providers.openai]
api_key = "sk-..."
model = "gpt-4o-mini"

[providers.ollama]
base_url = "http://localhost:11434"
model = "llama3.2:3b"
```

You can list multiple providers; gfix picks one per call based on precedence (Anthropic -> OpenAI -> Gemini -> Ollama), or honors `GITFIX_BYOK_PROVIDER=gemini` if set. Per-provider base-URL overrides via `GITFIX_BYOK_OPENAI_BASE_URL`, `GITFIX_BYOK_ANTHROPIC_BASE_URL`, `GITFIX_BYOK_OLLAMA_BASE_URL` — useful for proxies and OpenAI-compatible endpoints.

---

## 5. Verify the install

A tiny fixture-and-merge demo. Should take under 2 minutes end-to-end.

```sh
# 1. Version check
gfix --version
# -> gfix 0.1.0-alpha (keys.toml: /Users/you/Library/Application Support/dev.gitfix.gitfix/keys.toml)

# 2. Create a fixture repo with a guaranteed-to-conflict same-line edit on a .txt file
cd /tmp && rm -rf gfix-tour && mkdir gfix-tour && cd gfix-tour
git init -b main -q
git config user.email "you@local" && git config user.name "You"
git config commit.gpgsign false

echo "Status: under active development. See roadmap for milestones and timeline." > STATUS.txt
git add STATUS.txt && git commit -qm "base: add STATUS.txt"

git checkout -q -b feat/a
printf 'Status: in beta. We aim for stability by Q4. See roadmap.\n' > STATUS.txt
git commit -aqm "feat/a: beta status"

git checkout -q main && git checkout -q -b feat/b
printf 'Status: feature-complete, entering maintenance mode. See roadmap.\n' > STATUS.txt
git commit -aqm "feat/b: maintenance status"

git checkout -q main
```

`.txt` is intentional — Mergiraf's tree-sitter allowlist doesn't include it, so the conflict falls through to the text floor and produces a guaranteed same-line conflict on `STATUS.txt`.

### Drive it from Claude Code

In Claude Code, open the `/tmp/gfix-tour` directory and prompt:

> Use gfix to merge feat/a and feat/b into main. Use AI suggestion for any unresolved conflicts.

Claude Code will call (in order):

1. `gitfix_merge_preview` -> returns 1 unresolved conflict on `STATUS.txt`, with a `merge_id` like `m_<timestamp>_<short-hash>` and a `conflict_id`.
2. `gitfix_conflict_get` with `include_ai_suggestion: true` -> returns the conflict bodies + a Gemini suggestion with rationale and a confidence score (often low on contradictory updates — Gemini correctly refuses to fake certainty on a genuine ambiguity).
3. `gitfix_conflict_resolve` with `kind: "ai-suggestion"` -> accepts the proposal, logs to `.gitfix/state.json`.
4. `gitfix_merge_apply` -> finalizes to a 3-parent merge commit. Writes the audit ref.

### Inspect the audit ref

```sh
git log --all --oneline | head
# You should see a merge commit on main + a synthetic commit on refs/gitfix/audit/<merge_id>

# Read the audit JSON for the merge you just made
git show refs/gitfix/audit/$(git for-each-ref --format='%(refname:short)' 'refs/gitfix/audit/*' | tail -1):audit.json | head -40
```

You'll see a JSON blob with `merge_id`, `decisions` array (one entry per conflict, with `kind: ai_suggestion_accepted` and `actor: ai-agent:claude-code`), and the full final state.

### Verify rerere caching fired

```sh
git for-each-ref refs/gitfix/rerere/
# -> one ref per accepted resolution. The blake3 hash in the ref name is content-addressed.
```

Run the same merge on a fresh clone (or another fixture with identical conflict content) and `gitfix_merge_preview` will resolve `STATUS.txt` via rerere with 0 AI calls. This is the "AI is paid once, replayed forever" claim — verifiable on your own machine in 5 minutes.

---

## 6. Troubleshooting

### `gfix --version` hangs or says "command not found"

- **macOS Gatekeeper blocked it.** See §2 — `xattr -d com.apple.quarantine /opt/homebrew/bin/gfix`.
- **PATH issue.** `which gfix` should print the install path. If empty, the install dir isn't on your shell PATH. For Homebrew on Apple Silicon, ensure `/opt/homebrew/bin` is in PATH (default for fresh installs; absent on some inherited shell setups).

### Claude Code doesn't see gfix's tools after `claude mcp add`

- **Restart Claude Code.** MCP server registrations are loaded at session start.
- **You used `--scope local` (the default).** The registration is project-pinned. Re-run with `--scope user` (see §3).
- **You omitted `--`.** The server name got consumed as an `-e` value. Re-run with `--` before the name (see §3).

### AI suggestion never fires — gfix returns no suggestion

- **`GITFIX_BYOK=1` not set.** The MCP server env needs this to enable BYO-key. Add to the `claude mcp add` command and re-register.
- **keys.toml missing the section header.** `[providers.gemini]` (or whichever provider) is required at the top of the block. Without it, gfix parses the file and reads the provider block as empty — silently. Double-check the header literally, including the dot and the brackets.
- **Wrong keys.toml path on macOS.** `gfix --version` prints the path gfix actually reads. If you wrote to `~/.config/gitfix/keys.toml` on macOS, gfix reads `~/Library/Application Support/dev.gitfix.gitfix/keys.toml` — they're different files. Edit the one `--version` prints.

### `gitfix_merge_preview` resolved the conflict via Mergiraf when I wanted to test the AI path

Mergiraf is working as designed — its tree-sitter grammars handle many structural merges that would otherwise need AI. To force the AI path for a test fixture, use a file extension Mergiraf's allowlist doesn't cover (e.g., `.txt`, `.log`, `.md` in some cases). The demo in §5 uses `.txt` for exactly this reason.

### `.gitfix/state.json` is showing up in my commits

Known issue (Issue #9). Workaround: add `.gitfix/` to your repo's `.gitignore` before the first `gitfix_merge_apply`. The alpha.2 fix is auto-gitignore on first state write.

### Mergiraf isn't installed — does gfix still work?

Yes. Mergiraf is invoked as a subprocess via `std::process::Command::new("mergiraf")` (Mergiraf is GPL-3.0; gfix never links it). If it's not on PATH, gfix falls back gracefully to the text floor. To install Mergiraf:

```sh
brew install mergiraf
# or: cargo install --git https://codeberg.org/mergiraf/mergiraf
```

Then `mergiraf --version` should print a version, and gfix will route conflicts through it automatically.

---

## 7. Where to get help

- **[faq.md](faq.md)** — answers to the most common questions about gfix.
- **[capability-matrix.md](capability-matrix.md)** — what gfix can and can't do today, with concrete artifacts you can fetch and inspect.
- **[posts/2026-06-launch.md](posts/2026-06-launch.md)** — the M2 retrospective; the dogfood story behind alpha.1.
- **Install or binary bugs** -> [github.com/ameyypawar/gfix/issues](https://github.com/ameyypawar/gfix/issues)
- **Docs typos, missing platform notes, capability matrix fixes** -> [this repo's Issues](https://github.com/ameyypawar/gfix-docs/issues)
- **Security reports** -> ameyap007aaa@gmail.com (do not file public — coordinated disclosure preferred)
