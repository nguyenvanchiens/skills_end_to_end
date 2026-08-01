---
name: commit
description: Deprecated. The Conventional-Commits + Jira-ID spec moved into the gitlab-flow skill's "commit and push" trigger (one canonical home). Use only as a pointer if you typed /commit out of habit.
---

# commit — moved into `gitlab-flow`

> ⚠️ **Deprecated as a standalone skill.** The commit spec that used to live here now lives in **one canonical place**: the `gitlab-flow` skill, under the **"Commit and push"** trigger. Keeping two copies caused them to drift — this pointer removes the duplication and the second invocation model (the `/commit WRA-9` slash form).

## Use this instead

- **Trigger phrase** (not a slash command): type `commit and push` in a normal prompt.
  - Composes a Conventional-Commits message, commits **locally**, then **asks before pushing** (never auto-pushes).
  - Quick mode: `commit and push --quick`.
- **TASK-ID** is taken automatically from the current branch name (`feature/WRA-9-...` → `WRA-9`).

gitlab-flow's "Commit and push" trigger still covers everything that used to live here: the process (repo-state probe, partial-staging guard, atomic check) is inline in its `SKILL.md`, and the lookup tables — `.commit-scopes` allowlist, the 11 allowed types, footers (`Closes`/`Refs`/`BREAKING CHANGE`), Quick mode, WIP/Spike, and revert format — now live one hop away in [`skills/gitlab-flow/commit-reference.md`](../gitlab-flow/commit-reference.md). Nothing was lost in the move.

## Not using GitLab / `glab`?

You can still install `gitlab-flow` and use **only** the `commit and push` trigger. Committing is local and the push step is gated/optional, so `glab` is **not** required just to commit. The branch-naming and `review the last change` triggers also work without `glab`.

## If you want it gone entirely

This file is a deliberate pointer, not the spec. To remove the standalone skill completely, delete the `skills/commit/` directory and drop its row from the README — gitlab-flow does not depend on it.
