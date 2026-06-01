# gfix

> Mergiraf for the easy half. gfix for the hard half.

A merge conflict resolver for the agentic-coding era. Local CLI + MCP server. Mergiraf — invoked via subprocess to honor its GPL-3.0 boundary — handles the structural half. gfix handles the ~50% of hard semantic conflicts everyone else gives up on, with an honest AI ceiling: host-provided sampling when your MCP host advertises it, or BYO-key for Anthropic / OpenAI / Gemini / Ollama. Uses the host model's sampling when available — your tokens, your suggestions, your budget.

## Install

Three install paths, same binary. Pick whichever fits your setup.

```sh
brew install ameyypawar/gfix/gfix
```

```sh
curl -fsSL https://gfix.space/install | sh
```

```sh
npm install -g @gitfix/cli@alpha
```

Verify:

```sh
gfix --version
```

## 30-second quickstart

Wire `gfix` into Claude Code as an MCP server (cross-directory, user-scope):

```sh
claude mcp add --scope user -- gfix /opt/homebrew/bin/gfix mcp
```

Then from any chat in any repo:

> Use gfix to merge feat/auth and feat/billing into main.

The agent calls `gitfix_merge_preview`, gets back the conflict list, walks each one with `gitfix_conflict_get` + `gitfix_conflict_resolve`, then `gitfix_merge_apply` finalizes. Every accepted resolution lands in `refs/gitfix/rerere/<hash>` so the same conflict on the next branch resolves locally without an AI call.

Prefer the shell? The standalone CLI (`gfix merge`, `gfix resolve`, `gfix preview`, `gfix status`, `gfix log`, `gfix abort`, `gfix init`) shipped in v0.1.0-alpha.6. Install with `npm install -g @gitfix/cli@alpha` or `brew install ameyypawar/gfix/gfix` (both resolve to alpha.6). See [parallel-agents.md](parallel-agents.md) for the shell-first workflow.

Full walkthrough: [setup.md](setup.md).

Running multiple agents in parallel? See [parallel-agents.md](parallel-agents.md).

## Why is the source private?

`gfix` is a one-developer indie product in alpha. Closed source during the brand-formation window protects against fork-and-flip while the design surface is still moving fast. The 5-line EULA in [LICENSE.txt](LICENSE.txt) lets you install on any number of machines for any purpose including commercial use; you just can't redistribute the binary or reverse-engineer it. Pricing is free during alpha; the pricing decision is deferred to v1.0.

If "closed-source binary" is a deal-breaker for your environment (compliance review, security audit, regulated industry), source access on request to **ameyap007aaa@gmail.com** with a one-paragraph use case. For most users, the audit refs (see below) are the trust-substitute for source visibility.

## What gfix does well

- **Every merge leaves a pushable, immutable audit ref.** `refs/gitfix/audit/<merge_id>` is a synthetic git commit whose tree holds `audit.json` with the full final merge state. Inspect with `git show`. Share with `git push origin refs/gitfix/audit/*`. gfix never pushes — you do.
- **Accepted resolutions cached + replayed across machines.** `refs/gitfix/rerere/<blake3>` stores every accepted resolution as a content-addressed synthetic commit. Push to any git remote (GitHub, bare, self-hosted). Your teammate's `gfix` on the same conflict triple replays your answer in <100ms with zero AI calls.
- **Deterministic floor that never regresses.** Text merge -> Mergiraf subprocess -> AI. Mergiraf handles structural merges (TypeScript signature changes, Java method bodies, etc.) before AI is ever asked. The deterministic floor is the product; the AI is an accelerant.
- **AI-grade resolution on the half Mergiraf can't.** Independent research (merde.ai) puts LLM resolution of complex semantic conflicts around ~50% — that's the half gfix is built for. Suggestions arrive with rationale and confidence scores; nothing is auto-applied. Low-confidence suggestions are surfaced as low-confidence, not laundered into certainty. The 50% you accept gets cached by rerere and replays free across machines forever — so the AI is paid once and reused.
- **Untrusted-content safe.** Conflict bodies are wrapped and defanged before being concatenated into the AI prompt. A jailbroken model can produce file bytes — never actions. The red-team corpus that found and fixed this bug is part of the M2 release.
- **MCP-native, host-agnostic.** Works with Claude Code today. Cursor, Codex, Aider, opencode, Continue, Gemini CLI — anything that speaks MCP — get the same tool surface for free.

## What gfix doesn't do

The honest counterpart. Closed-source means honesty is the trust layer; here's the explicit list, not buried elsewhere.

- **Doesn't prevent conflicts before they happen.** gfix is a resolver, not a coordinator. If you want entity-level prevention, see [Weave](https://ataraxy-labs.github.io/weave/) (open source, MIT/Apache).
- **Doesn't push to remotes.** By design — gfix writes locally, you push `refs/gitfix/*` when you want to share them. Merges into `main` require `auto_approve=true`.
- **Doesn't auto-merge outside Mergiraf's language allowlist.** Anything outside falls through to the text floor → AI-suggestion fallback. No silent skip; the AI path is language-agnostic.
- **~50% hard-conflict resolution; the other 50% escalates.** Independent research (merde.ai) puts LLM resolution of complex semantic conflicts at ~50%. Rerere amortizes the half that worked; the half that didn't surfaces honestly.
- **Doesn't auto-apply AI output.** Every accepted ai-suggestion requires an explicit `gitfix_conflict_resolve` call with `kind: "ai-suggestion"`. gfix never silently writes model output.

## Where things live

- **This repo (`gfix-docs`)** — user-facing docs, setup guide, capability matrix, FAQ, changelog, launch posts. The repo you're in.
- **[`ameyypawar/gfix`](https://github.com/ameyypawar/gfix)** — binary distribution surface. GitHub Releases host the actual tarballs. Install bugs go here.
- **`ameyypawar/core-merge-engine` source** — PRIVATE. Closed during alpha. Source access on request (see above).

## Status

**v0.1.0-alpha** — closed-source binary alpha. Free during alpha. Pricing decision deferred to v1.0.

**Supported platforms:** macOS arm64 (verified by developer dogfood across two sessions). macOS x86_64, Linux x86_64, Linux arm64 — binaries shipped, unverified by the developer's own daily use.

**Supported MCP host:** Claude Code (canary). Cursor, Codex, Aider, opencode, Continue, Gemini CLI — should work, not actively tested against in alpha.1.

## Reporting issues

- **Install or binary bugs** -> [github.com/ameyypawar/gfix/issues](https://github.com/ameyypawar/gfix/issues)
- **Docs typos, missing platform notes, capability matrix fixes** -> [this repo's Issues](https://github.com/ameyypawar/gfix-docs/issues)
- **Security reports** -> ameyap007aaa@gmail.com (do not file public — coordinated disclosure preferred)

## License

Proprietary alpha. The full 5-line EULA lives in [LICENSE.txt](LICENSE.txt). Short version: install on any number of machines for any purpose including commercial use; don't redistribute the binary or reverse-engineer it. A formal EULA supersedes this notice before v1.0.

```
gfix end-user license — v0.1.0-alpha

Copyright © 2026 Amey Pawar. All rights reserved.

You may install and use this binary on any number of machines for any
purpose, including commercial use. You may not (a) reverse-engineer or
attempt to extract source code from the binary, (b) redistribute the
binary or any portion of it, (c) remove this notice, or (d) use it to
develop a competing product. This software is provided as-is, with no
warranty. A formal EULA will supersede this notice before v1.0.

For licensing questions: ameyap007aaa@gmail.com
```
