---
name: review-branch
description: Review the cumulative changes of the current branch against main (committed + uncommitted) for reuse, quality, and efficiency, then fix any issues found. Use when finishing a feature branch before opening an MR.
---

# Review Branch: Code Review and Cleanup

Review every change the current branch introduces relative to `main` — committed AND uncommitted — for reuse, quality, and efficiency. Fix issues directly.

## Phase 1: Identify Changes

1. Resolve the merge base against `main`:

   ```bash
   git merge-base main HEAD
   ```

2. If the current branch IS `main` (or no merge base differs from HEAD), tell the user there is nothing to review and stop.

3. Capture the cumulative diff (branch commits + working tree) into a temp file so the agents can read it without flooding your context:

   ```bash
   BASE=$(git merge-base main HEAD)
   git diff --no-color "$BASE" > /tmp/review_branch.diff
   wc -l /tmp/review_branch.diff
   ```

4. Capture the list of new (untracked) files and their contents — `git diff` does not include them:

   ```bash
   git ls-files --others --exclude-standard > /tmp/review_branch_new.txt
   ```

5. Also capture the list of files changed (committed-only summary) so you can spot-check:

   ```bash
   git diff --stat "$BASE"
   ```

## Phase 2: Launch Three Review Agents in Parallel

Use the Agent tool to launch all three agents concurrently in a single message. Pass each agent:
- The diff path (`/tmp/review_branch.diff`)
- The new-files path (`/tmp/review_branch_new.txt`) so they read each new file in full
- The merge base context ("This is the cumulative diff of branch <name> against main")

### Agent 1: Code Reuse Review

For each change:

1. **Search for existing utilities and helpers** that could replace newly written code. Look for similar patterns elsewhere in the codebase — common locations are utility directories, shared modules, and files adjacent to the changed ones.
2. **Flag any new function that duplicates existing functionality.** Suggest the existing function to use instead.
3. **Flag any inline logic that could use an existing utility** — hand-rolled string manipulation, manual path handling, custom environment checks, ad-hoc type guards, and similar patterns are common candidates.

### Agent 2: Code Quality Review

Review the same changes for hacky patterns:

1. **Redundant state**: state that duplicates existing state, cached values that could be derived, observers/effects that could be direct calls
2. **Parameter sprawl**: adding new parameters to a function instead of generalizing or restructuring existing ones
3. **Copy-paste with slight variation**: near-duplicate code blocks that should be unified with a shared abstraction
4. **Leaky abstractions**: exposing internal details that should be encapsulated, or breaking existing abstraction boundaries
5. **Stringly-typed code**: using raw strings where constants, enums (string unions), or branded types already exist in the codebase
6. **Unnecessary JSX nesting**: wrapper Boxes/elements that add no layout value — check if inner component props (flexShrink, alignItems, etc.) already provide the needed behavior
7. **Nested conditionals**: ternary chains (`a ? x : b ? y : ...`), nested if/else, or nested switch 3+ levels deep — flatten with early returns, guard clauses, a lookup table, or an if/else-if cascade
8. **Unnecessary comments**: comments explaining WHAT the code does (well-named identifiers already do that), narrating the change, or referencing the task/caller — delete; keep only non-obvious WHY (hidden constraints, subtle invariants, workarounds)

### Agent 3: Efficiency Review

Review the same changes for efficiency:

1. **Unnecessary work**: redundant computations, repeated file reads, duplicate network/API calls, N+1 patterns
2. **Missed concurrency**: independent operations run sequentially when they could run in parallel
3. **Hot-path bloat**: new blocking work added to startup or per-request/per-render hot paths
4. **Recurring no-op updates**: state/store updates inside polling loops, intervals, or event handlers that fire unconditionally — add a change-detection guard so downstream consumers aren't notified when nothing changed
5. **Unnecessary existence checks**: pre-checking file/resource existence before operating (TOCTOU anti-pattern) — operate directly and handle the error
6. **Memory**: unbounded data structures, missing cleanup, event listener leaks
7. **Overly broad operations**: reading entire files when only a portion is needed, loading all items when filtering for one

## Phase 3: Fix Issues

Wait for all three agents to complete. Aggregate their findings and fix each issue directly. If a finding is a false positive or not worth addressing, note it and move on — do not argue with the finding, just skip it.

When done, briefly summarize what was fixed (or confirm the code was already clean) and the test/typecheck status. Do NOT commit or push — leave the cleanup as working-tree changes for the user to review and commit.

## Notes

- This reviews the WHOLE branch's contribution, not just the latest commit. Use `/simplify` for the narrower "review uncommitted changes only" workflow.
- If the diff is very large (>2000 lines), warn the user that the agent reviews may be coarse-grained, and suggest running the skill earlier next time (e.g. after each meaningful commit).
- Determining "main": this skill assumes the branch's integration target is named `main`. If the repo uses `master` or `develop`, ask the user once, then use that.
