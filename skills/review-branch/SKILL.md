---
name: review-branch
description: Use when finishing a feature branch before opening an MR, or when reviewing everything a branch introduced against its base branch (committed + uncommitted) rather than only the latest commit.
---

# Review Branch: Code Review and Cleanup

Review every change the current branch introduces relative to its base branch `<base>` — committed AND uncommitted — through four specialized lenses: correctness against the task, security, efficiency, and quality/reuse. Verify the findings, then fix the ones that survive.

**Core principle:** the quality of the review depends on the context the reviewer loaded, not on how many agents you launch. An agent that only sees the diff will confidently report bugs that do not exist and miss the ones that do.

## Phase 1: Identify Changes

1. **Determine the base branch `<base>` — do NOT assume `main`.** Run `git branch -r` to see the real integration branches on the remote, then:
   Resolve `<base>` in this priority order (first match wins):
   1. **Current branch has an open MR?** Run `glab mr view` (no arg → the MR of the *current* branch; works after `glab mr checkout <N>` or on any branch whose MR exists). If it returns an MR, use its `targetBranch` as `<base>` and don't ask — the MR declares what it merges into (`main`-primary repos → `main`, `dev`-primary repos → `dev`). If it returns nothing / errors (repo doesn't use MRs, or none opened yet), fall through.
   2. **User named the base in the prompt** (e.g. "review against dev") → use it, don't ask.
   3. **`<base>` already established for this branch earlier in the session** (e.g. by gitlab-flow when the branch was created) → reuse it.
   4. **Otherwise** → ask the user once: "Review against which base — `main` or `dev`?" (offer the names you actually saw via `git branch -r`, e.g. `develop`/`master`).

2. Resolve the merge base against `<base>`:

   ```bash
   git merge-base <base> HEAD
   ```

3. If the current branch IS `<base>` (or no merge base differs from HEAD), tell the user there is nothing to review and stop.

4. Capture the cumulative diff (branch commits + working tree) into a temp file so the agents can read it without flooding your context. Write it inside `.git/` (always present, writable, never committed) — do **not** use `/tmp/`, which does not exist reliably on Windows/PowerShell:

   ```bash
   BASE=$(git merge-base <base> HEAD)
   git diff --no-color "$BASE" > .git/review_branch.diff
   wc -l .git/review_branch.diff
   ```

5. Capture the list of new (untracked) files and their contents — `git diff` does not include them:

   ```bash
   git ls-files --others --exclude-standard > .git/review_branch_new.txt
   ```

6. Also capture the list of files changed (committed-only summary) so you can spot-check:

   ```bash
   git diff --stat "$BASE"
   ```

7. **Load the task context.** Without it, Agent 1 below has no standard to judge "correct" against and degrades into a style checker.
   - Extract the task ID from the branch name (pattern `[A-Z][A-Z0-9]+-\d+`) if present.
   - Get the task description, stopping at the first hit: already stated in this conversation → the MR description if one is open (`glab mr view`) → **ask the user one short question**: "What does this branch need to accomplish? (1-2 sentences)".
   - If the user declines, still run Phase 2 but **say so explicitly**: "no task context — Agent 1 can only check self-consistency, not whether the logic matches the requirement".

## Phase 2: Launch Four Review Agents in Parallel

Use the Agent tool to launch all four agents concurrently in a single message. Pass each agent:
- The diff path (`.git/review_branch.diff`)
- The new-files path (`.git/review_branch_new.txt`) so they read each new file in full
- The merge base context ("This is the cumulative diff of branch <name> against <base>")
- The task description from Phase 1 step 7

**Copy both blocks below verbatim into all four agent prompts.** Subagents run in their own context — they cannot see anything else in this file.

**Block 1 — Severity:**

> Assign a severity to every finding: `Blocker` (wrong logic vs the task, security hole, data loss, crash) · `Major` (edge case that will realistically happen, N+1 on a hot path, race condition) · `Minor` (naming, dead code, clumsy abstraction) · `Nit` (style, personal preference — drop these, do not report them). Every finding needs a `file:line` plus the code that proves it. **If you find no Blocker or Major, return an empty list — do not pad the list to look thorough.**

**Block 2 — Grounding.** This is what separates a real review from guessing at a diff:

> **Before flagging anything, load context — never review a diff through a straw:**
> 1. For EVERY file in the diff: **read the full file**, not just the hunk. The diff cuts away the very code that decides whether a finding is real.
> 2. Read `CLAUDE.md` (if present) plus 1-2 files in the same directory/module as the changes, to learn the repo's **actual** conventions. Do not apply generic ones.
> 3. If a function, API, or schema changed signature or behavior: `Grep` for **every caller**, including files not in the diff. A change that looks safe inside the diff can still break a caller elsewhere — this is exactly the class of bug a diff-only review never sees.
> 4. Only flag after doing 1-3. Findings inferred from a diff alone are the #1 source of false positives.

### Agent 1: Correctness / Task-fit Review

Judge the change against the task description, then against itself:

1. **Requirement drift**: behavior that does not match the stated task, cases named in the task that are not handled, work done beyond the task's scope
2. **Edge cases**: null/undefined, empty string, empty list, boundary values, negative numbers, unicode, duplicate input, very large input
3. **Error handling**: `catch` blocks that swallow the error, errors logged but execution continuing as if nothing happened, missing rollback
4. **Partial state**: a throw halfway through a multi-step mutation leaving the system in a state no code path expects
5. **Off-by-one, timezone, and rounding** errors in date, pagination, and money arithmetic
6. **Non-idempotent migrations or jobs** that corrupt data when re-run

### Agent 2: Security Review

1. **Missing input validation** on anything crossing a trust boundary
2. **Authz/authn bypass**: a check enforced in the UI but not in the API, missing ownership check on an object fetched by ID
3. **Injection**: SQL, command, template, path traversal
4. **Mass assignment**: binding a whole request body onto a model, letting a caller set fields they should not
5. **Secret and PII exposure**: tokens, keys, passwords, or personal data reaching logs, error responses, or the client
6. **SSRF** on any server-side fetch built from user input
7. **Transport and session misconfiguration**: over-broad CORS, cookies missing `HttpOnly`/`Secure`/`SameSite`

### Agent 3: Efficiency Review

1. **Unnecessary work**: redundant computations, repeated file reads, duplicate network/API calls, N+1 patterns
2. **Missed concurrency**: independent operations run sequentially when they could run in parallel
3. **Hot-path bloat**: new blocking work added to startup or per-request/per-render hot paths
4. **Recurring no-op updates**: state/store updates inside polling loops, intervals, or event handlers that fire unconditionally — add a change-detection guard so downstream consumers aren't notified when nothing changed
5. **Unnecessary existence checks**: pre-checking file/resource existence before operating (TOCTOU anti-pattern) — operate directly and handle the error
6. **Memory**: unbounded data structures, missing cleanup, event listener leaks
7. **Overly broad operations**: reading entire files when only a portion is needed, loading all items when filtering for one

### Agent 4: Quality & Reuse Review

1. **Search for existing utilities and helpers** that could replace newly written code. Look for similar patterns elsewhere in the codebase — common locations are utility directories, shared modules, and files adjacent to the changed ones.
2. **Flag any new function that duplicates existing functionality.** Suggest the existing function to use instead.
3. **Flag any inline logic that could use an existing utility** — hand-rolled string manipulation, manual path handling, custom environment checks, ad-hoc type guards, and similar patterns are common candidates.
4. **Redundant state**: state that duplicates existing state, cached values that could be derived, observers/effects that could be direct calls
5. **Parameter sprawl**: adding new parameters to a function instead of generalizing or restructuring existing ones
6. **Copy-paste with slight variation**: near-duplicate code blocks that should be unified with a shared abstraction
7. **Leaky abstractions**: exposing internal details that should be encapsulated, or breaking existing abstraction boundaries
8. **Stringly-typed code**: using raw strings where constants, enums (string unions), or branded types already exist in the codebase
9. **Unnecessary JSX nesting**: wrapper Boxes/elements that add no layout value — check if inner component props (flexShrink, alignItems, etc.) already provide the needed behavior
10. **Nested conditionals**: ternary chains (`a ? x : b ? y : ...`), nested if/else, or nested switch 3+ levels deep — flatten with early returns, guard clauses, a lookup table, or an if/else-if cascade
11. **Unnecessary comments**: comments explaining WHAT the code does (well-named identifiers already do that), narrating the change, or referencing the task/caller — delete; keep only non-obvious WHY (hidden constraints, subtle invariants, workarounds)

> **Why this roster**: `Blocker` means wrong logic, a security hole, data loss, or a crash — so Agents 1 and 2 own the only category that actually blocks a merge, and they are the two you must not drop. Agent 4 folds reuse and quality into one slot because its output is almost entirely `Minor`, which never gets auto-fixed.

## Phase 2.5: Dedup and Verify

Do NOT concatenate the four lists and start fixing. Each agent ran in its own context through a single lens, so the combined list contains duplicates and findings that are simply wrong. Phase 3 edits real code from this list, so a bad entry here costs more than it would in a read-only review.

1. **Dedup** by `file:line` and by underlying problem. Keep the clearest description and note how many agents reported it — **two independent agents pointing at the same line is a strong signal**, handle those first.
2. **Verify every `Blocker` and `Major`.** Default to **rejecting** the finding; keep it only if all three hold:
   - Open the real file at `file:line` and read the surrounding code. Does the problem exist in the current code, or did the agent misread a hunk?
   - Is the buggy path **reachable**? (dead branch, caller already guards it, input validated at a higher layer → reject)
   - Does the suggested fix actually apply to this repo? (does the helper or utility the agent proposed really exist?)
   - Any of these unproven → **drop it from the auto-fix list**. If it still seems worth a look, move it to a "needs confirmation" section for the user instead of fixing it.
3. `Minor` findings only need dedup, not verification — nothing auto-fixes them, so a wrong one cannot damage the code.

The verified list will be much shorter than the four raw lists. That is the correct outcome, not a weak review.

## Phase 3: Fix and Report

1. Fix only the `Blocker` and `Major` findings that survived Phase 2.5. Collect `Minor` into its own section for the user to decide on; drop `Nit` entirely. Nothing left after verification → say "no blocking issues found" and stop. An empty list is a valid result — do not promote a `Minor` to fill it out.
2. Do NOT commit or push — leave the changes in the working tree for the user to review and commit.
3. Summarize in this shape:

   ```
   Fixed: <N> Blocker, <M> Major — <files>
   Verify: dropped <K> unverifiable findings (<T> raw findings from 4 agents)
   Test/typecheck: <status>

   ### Minor (not fixed — your call)
   #1 [Minor] path/file.ts:15 — <problem>

   ### Needs confirmation (not fixed)
   #2 path/file.ts:80 — <problem>. Unproven because <reason>
   ```

## Notes

- This reviews the WHOLE branch's contribution, not just the latest commit. Use `/simplify` for the narrower "review uncommitted changes only" workflow.
- If the diff is very large (>2000 lines), warn the user that the agent reviews may be coarse-grained, and suggest running the skill earlier next time (e.g. after each meaningful commit).
- Determining `<base>`: see Phase 1 step 1 — never assume `main`. Ask the user (`main` vs `dev`/`develop`/`master`) unless they named it in the prompt or it was already established earlier in the session. Once chosen, use the same `<base>` for the merge base and all comparisons.
