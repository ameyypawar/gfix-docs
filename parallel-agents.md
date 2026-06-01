# Parallel agents with git worktrees

Running several coding agents at once means several branches in flight. This is the workflow I actually use — one agent per git worktree, non-overlapping slices, and gfix where the lanes collide.

Path through this doc:
1. [Why worktrees](#1-why-worktrees)
2. [Setup: one worktree per lane](#2-setup-one-worktree-per-lane)
3. [Partitioning to minimize collisions](#3-partitioning-to-minimize-collisions)
4. [When the lanes meet: resolving with gfix](#4-when-the-lanes-meet-resolving-with-gfix)
5. [Worked example: four lanes clearing a backlog](#5-worked-example-four-lanes-clearing-a-backlog)
6. [Cleanup](#6-cleanup)

---

## 1. Why worktrees

A git worktree gives you an isolated working tree checked out from the same `.git` directory — no separate clone, no duplicated object store. Each worktree has its own branch, its own index, and its own working files. Three things this arrangement buys you:

- **One object store, four simultaneous branches.** `git worktree add` is cheap: it checks out a branch into a new directory, shares the pack files, and keeps one reflog. No clone overhead.
- **The shared `.git` is why gfix works across lanes.** gfix writes rerere refs to `refs/gitfix/rerere/<blake3-hash>` and audit refs to `refs/gitfix/audit/<merge_id>` in the repo's ref namespace. Because all worktrees share one `.git`, a resolution accepted in the first fold — lane-b into lane-a — is already available when lane-c folds in. No push, no copy, no extra step. The shared ref namespace is the amortization surface.
- **Agents stay independent.** Each worktree directory is a self-contained working tree. An agent editing files in `../proj-wt2` cannot interfere with files in `../proj-wt1` — the working trees are separate even though the repo is shared.

---

## 2. Setup: one worktree per lane

Start from a repo that has a stable `main` (or any base you want to merge back into). Branch all lanes from the same commit — this is what makes the eventual merge a clean two-way comparison rather than a three-way merge against diverged histories.

```sh
git worktree add ../proj-wt1 -b lane-a origin/main
git worktree add ../proj-wt2 -b lane-b origin/main
git worktree add ../proj-wt3 -b lane-c origin/main
git worktree add ../proj-wt4 -b lane-d origin/main

git worktree list
```

Point each agent at its worktree directory. `../proj-wt1` is the root for the agent working on `lane-a`, `../proj-wt2` for `lane-b`, and so on.

One lane per worktree. One agent per lane. The agent in `../proj-wt2` runs `git add`, `git commit`, `git diff` — all scoped to its working tree and branch. It cannot see `lane-a`'s uncommitted changes.

---

## 3. Partitioning to minimize collisions

Assign each lane a non-overlapping slice of the work before the agents start. The partitioning can be by file path (lane-a owns `crates/auth/`, lane-b owns `crates/billing/`) or by issue/task (lane-a closes issues #12 and #15, lane-b closes #18 and #22). Either way, the goal is to make the seam between lanes as thin as possible.

You can't fully avoid overlap and you shouldn't burn time chasing zero collisions. Shared types, a router that wires every subsystem, a barrel export — these touch every lane by definition. The practical rule: partition to make collisions **rare and small**, let gfix handle the residue. A collision that surfaces as one or two same-function edits takes seconds to resolve. Collisions that surface as competing rewrites of a 300-line module cost more — partition to avoid those specifically.

One heuristic: look at the files that changed most in the last 30 days. The hot files are the ones multiple lanes will want to touch. Either assign each hot file to exactly one lane, or accept that it'll be a conflict and make sure it's on the radar.

gfix is the resolver at the seam, not a coordinator that prevents collisions before they happen. For the coordination layer, see [capability-matrix.md](capability-matrix.md) — the "not an orchestrator" row is the relevant boundary.

---

## 4. When the lanes meet: resolving with gfix

When two lanes have edited the same lines, gfix runs a layered resolution stack:

1. **Text merge** — fast, deterministic. Same-region edits that don't conflict (e.g., A adds a function at line 50, B adds a different function at line 80) resolve here with no AI involvement.
2. **Mergiraf subprocess** — for languages with tree-sitter grammars (Rust, TypeScript, JavaScript, Python, Go, Java, and others), Mergiraf resolves structural conflicts that look like line-level collisions but are actually reconcilable at the AST level. gfix invokes Mergiraf as a subprocess — Mergiraf is GPL-3.0; gfix never links it. If Mergiraf isn't installed, gfix falls back to the text floor gracefully.
3. **AI suggestion** — the semantic remainder. Conflicts that are genuinely ambiguous — two agents with competing intent on the same code — go here. Suggestions arrive with rationale and a confidence score; nothing is auto-applied.

**Rerere: the reason it scales.** Every accepted resolution (except Mergiraf-deterministic and rerere-replay-itself) is written to `refs/gitfix/rerere/<blake3-hash>` as a synthetic git commit. The hash is content-addressed over `(file_path, base_oid, ours_oid, theirs_oid)`. The next merge with the same conflict triple — whether it's folding the next lane, running on a teammate's machine, or re-running after a rebase — auto-replays the accepted answer in <100ms with zero AI calls.

Because all worktrees share one `.git`, the rerere ref written during the first lane fold is visible to every subsequent fold without any extra push or fetch. Resolve once. Replay free.

Share across machines with:

```sh
git push origin refs/gitfix/rerere/*
```

gfix never pushes. You do.

---

### CLI path

Install alpha.6 or later:

```sh
npm install -g @gitfix/cli@alpha
# or
brew install ameyypawar/gfix/gfix
```

Run from the integration branch (the branch you're folding into — `lane-a` in this example):

```sh
gfix merge lane-b --target lane-a
```

gfix returns a conflict list. Unresolved conflicts stay unresolved until you explicitly resolve each one:

```sh
gfix resolve <conflict_id> --via ours|theirs|take-target|mergiraf|ai-suggestion|manual
```

When all conflicts are resolved, re-run `gfix merge lane-b --target lane-a` to finalize and commit.

---

### MCP path (agent-driven)

For agents running in Claude Code, Cursor, or any MCP host, the tool sequence mirrors the CLI:

1. `gitfix_merge_preview` -> returns the conflict list with `merge_id` and per-conflict `conflict_id`s. Resolved conflicts show their resolution kind; unresolved ones need a decision.
2. `gitfix_conflict_get` with `include_ai_suggestion: true` -> returns the full `ours`/`theirs`/`base`/`target` bodies for a specific conflict, plus an AI suggestion (cached on the merge state for the resolve call).
3. `gitfix_conflict_resolve` with `kind: "ai-suggestion"` (or `"ours"`, `"theirs"`, `"mergiraf"`, `"manual"`) -> accepts one resolution. Records to `.gitfix/state.json`.
4. `gitfix_merge_apply` -> finalizes: materializes all accepted resolutions, writes the local merge commit, writes the audit ref at `refs/gitfix/audit/<merge_id>`.

For a full walkthrough of the MCP path in isolation, see [setup.md §5](setup.md#5-verify-the-install).

---

### VS Code

The `gitfix` extension (available on the Marketplace and Open VSX) surfaces the in-flight merge state as a review panel — conflict bodies side-by-side, resolution kind visible per conflict, audit ref accessible after apply. Useful for human review during or after a merge. The extension calls the same MCP tools; it's a review surface, not a separate resolution engine.

---

## 5. Worked example: four lanes clearing a backlog

Four worktrees, four lanes, one integration branch. The folding order is serial — lane-b into lane-a, then lane-c into the result, then lane-d — so each individual merge is two-way. Rerere carries accepted fixes forward across folds without re-running AI.

**Setup.** Four agents ran in parallel across `../proj-wt1` through `../proj-wt4`, each closing a slice of open issues. All branched from the same base commit on `origin/main`. Each lane is clean and pushed to its remote branch.

**Fold 1: lane-b into lane-a.**

```sh
$ gfix merge lane-b --target lane-a
merge  m_2026-...
target lane-a
sources lane-b

resolved  (4)
  crates/.../access_control.rs  via text
  crates/.../rerere.rs          via text
  crates/.../state.rs           via text
  crates/.../mcp/server.rs      via mergiraf

unresolved  (1)
  conflict_id  file                kind  ours    theirs
  c_78e2d      .../substrate.rs    ast   lane-a  lane-b

Unresolved conflicts remain. Resolve each with:
  gfix resolve <conflict_id> --via ours|theirs|mergiraf|manual
(exits non-zero)
```

Four conflicts resolved automatically — three by text merge, one by Mergiraf's tree-sitter grammar for Rust. One AST-level conflict in `substrate.rs` needs a decision. Here we take `lane-a`'s version:

```sh
$ gfix resolve c_78e2d --via ours
resolved c_78e2d via ours  (0 conflict(s) remaining)
```

Finalize:

```sh
$ gfix merge lane-b --target lane-a
merge m_2026-...: committed  head eeec2f1
```

Audit ref written at `refs/gitfix/audit/m_2026-...`. Rerere ref written at `refs/gitfix/rerere/<blake3>` for the `substrate.rs` resolution.

Build and tests pass. Move to the next fold.

**Fold 2: lane-c into lane-a.**

`substrate.rs` was also edited by lane-c — same base, same conflict shape. gfix replays the rerere ref from fold 1 automatically. No AI call. The conflict_id for `substrate.rs` resolves via rerere in the preview step; only new conflicts (if any) need decisions.

**Fold 3: lane-d into lane-a.**

Same pattern. Rerere carries the `substrate.rs` resolution forward again. Build and tests pass.

**Final state.** `lane-a` is the integration branch — it holds the merged work of all four lanes, with four audit refs recording every decision, and rerere refs that can be pushed to the team remote for any future merge against this same conflict surface.

---

## 6. Cleanup

When a lane is fully merged, remove its worktree:

```sh
git worktree remove ../proj-wt1
git worktree prune
```

`git worktree remove` removes the working tree directory. It does **not** delete the branch or its commits — those remain in the repo history under `lane-a`, `lane-b`, etc. If you want to delete the branches too, do it explicitly:

```sh
git branch -d lane-b lane-c lane-d
```

Audit refs at `refs/gitfix/audit/*` and rerere refs at `refs/gitfix/rerere/*` persist regardless of worktree or branch deletion. They live in the ref namespace, not in the working tree, and travel with the repo when pushed.

---

## Where to go next

- [setup.md](setup.md) — install, wire gfix into your MCP host, configure a BYO-key provider.
- [faq.md](faq.md#how-do-i-share-rerere-across-my-team) — sharing rerere refs across teammates; cross-machine replay.
- [capability-matrix.md](capability-matrix.md) — what gfix can and can't do today, with concrete artifacts from the dogfood sessions.
