---
name: commit
description: Commit staged + working-tree changes following Conventional Commits, with the Jira ID appended in parentheses at the end of the subject line. Takes the Jira ID as an argument, e.g. `/commit WRA-9`.
---

# Commit — Conventional Commits + Jira ID

Spec: <https://www.conventionalcommits.org/en/v1.0.0/>

## Inputs

| Input | Rule |
|-------|------|
| `args` | First token = Jira ID. Empty → STOP, ask user. |
| Jira ID pattern | `[A-Z][A-Z0-9]+-\d+` (e.g. `WRA-46`, `IT-12141`, `ERP-3079`) |
| `--quick` flag | If present in `args` → see [Quick mode](#quick-mode) |
| Repo language | Vietnamese (per `git log`) — applies to `subject` + `body` |
| Meta-rule | Editing this skill itself uses the same skill: `chore(skill): … (WRA-XX)` — no exception |

## Behavior

| Rule | Detail |
|------|--------|
| Probe before deciding | Always run Step 1 in full — don't skip even for trivial-looking commits |
| Never guess `type` / `scope` | Uncertain → STOP, ask user. No coin-flip, no guessing |
| Quality > speed | One confirmation question beats one wrong-shaped commit |
| Detect "quick commit" intent | User says "nhanh" / "quick" / "tạm" / "small" / "fast" → suggest `--quick` before committing |

## Process

### Step 1 — Probe repo state (parallel calls in one message)

| Call | Purpose |
|------|---------|
| `git status` (no `-uall`) | Untracked + modified files |
| `git diff --cached` | Staged hunks only |
| `git diff` | Unstaged hunks only — separated to detect partial-staging |
| `cat "$(git rev-parse --show-toplevel)/.commit-scopes"` | Scope allowlist (works from any subdir) |

### Step 2 — Partial-staging guard

| When | Action |
|------|--------|
| File appears in both index and worktree (`MM` in `git status`) | STOP, ask user |
| User: commit staged-only | Proceed with current index |
| User: stage rest then combine | `git add <files>` then commit |
| Default | Never auto-`git add` unstaged hunks (user may have intentionally `git add -p`'d) |

### Step 3 — Atomic check

| When | Action |
|------|--------|
| Single logical change spanning N modules (e.g. add field: migration + model + API + UI) | Atomic — 1 commit, OK |
| ≥2 unrelated modules/scopes | STOP, ask user |
| User: split | Stage per group → separate commits, each with own `type`/`scope` |
| User: combine | Drop `(<scope>)` — never use `core`/`misc` filler |
| User wants single commit but multi-type | Pick `type` reflecting dominant change |

Heuristic: removing one module breaks feature → atomic. Standalone meaningful → split.

### Step 4 — Compose message

| Part | Rule |
|------|------|
| Format | `<type>(<scope>): <subject> (<JIRA-ID>)` |
| Jira ID position | End of subject, in `()`, exactly one occurrence |
| Header length | ≤ 100 chars total (target ≤ 72) |
| `type` / `scope` | English (CC standard) |
| `scope` | From `.commit-scopes` (see [Scope](#scope)). Drop `(<scope>)` if change spans modules |
| `subject` | Imperative, no period, lowercase first letter |
| `subject` exceptions | Acronyms uppercase: `JWT`, `API`, `OIDC`, `VAT`. Proper nouns: `Jira`, `Redis`, `GitLab` |
| `body` | Optional. Wrap at 72 chars. Why > what. Single-level bullets only |
| Breaking change | Add `!` after `type(scope)` (e.g. `feat(api)!:`) + footer `BREAKING CHANGE: <desc>` |

### Step 5 — Commit (HEREDOC / here-string)

**Bash (Claude Code Bash tool):**

```bash
# With scope
git commit -m "$(cat <<'EOF'
<type>(<scope>): <subject> (<JIRA-ID>)

<body optional>
EOF
)"

# Without scope
git commit -m "$(cat <<'EOF'
<type>: <subject> (<JIRA-ID>)

<body optional>
EOF
)"
```

> No `Co-Authored-By:` auto-insert — repo doesn't track AI co-authorship. Real pair-coding → use `Co-authored-by` footer (see [Footer](#footer)).

**PowerShell (user copy-paste on Windows terminal):**

```powershell
# Single-quoted here-string — no $variable expansion
git commit -m @'
<type>(<scope>): <subject> (<JIRA-ID>)

<body optional>
'@
```

> Closing `'@` must be at column 0 — indent = parse error.

## Allowed types

| Type | Meaning | Version bump |
|------|---------|--------------|
| `feat` | New feature | MINOR (1.X.0) |
| `fix` | Bug fix | PATCH (1.0.X) |
| `perf` | Performance improvement | PATCH |
| `refactor` | Refactor without behavior change | — |
| `docs` | Documentation | — |
| `test` | Add/modify tests | — |
| `build` | Build system, dependency, packaging | — |
| `style` | Format code (whitespace, lint) — no behavior change | — |
| `chore` | Maintenance (housekeeping, fits no other type) | — |
| `ci` | CI/CD config | — |
| `revert` | Revert prior commit (see [Examples](#examples) — revert) | — |

> **Breaking change** is a *modifier*, not a separate type. Suffix `!` (e.g. `feat(api)!:`) or footer `BREAKING CHANGE:` → MAJOR bump (X.0.0).

## Footer

Position: after body, separated by blank line. Format: `Token: value` (CC) or `Token #issue` (GitHub-style).

| Footer | When to use |
|--------|-------------|
| `BREAKING CHANGE: <desc>` | Required when header has `!`. Describe impact + migration. |
| `Closes <JIRA-ID>` | Trigger Jira-GitLab auto-close on merge. Skip if already auto-closing from subject mention (check 1-2 recent merged tickets to confirm) — avoids double-trigger. |
| `Refs <JIRA-ID>` | Reference another Jira (related but not closing). |
| `Co-authored-by: Name <email>` | Real pair-programming. Never auto-insert AI here. |
| `Reviewed-by: Name <email>` | Optional — only if team convention. |

| Rule | When to apply |
|------|---------------|
| Don't repeat Jira ID | If already in subject `(<JIRA-ID>)`, omit from footer unless `Closes`/`Refs` keyword needed |
| Token case | PascalCase or kebab-case (`Reviewed-by`, `Co-authored-by`); `BREAKING CHANGE` uppercase per spec |
| Multi-line value | Indent continuation lines (CC parser detects via indent). Prefer single-line. |

## Scope

**Lookup**: read `.commit-scopes` at repo root → fallback `git log --pretty=format:%s | grep -oE '\([^)]+\):' | sort -u`.

| Convention | Detail |
|------------|--------|
| Case | lowercase, kebab-case (separator `-`, not `_`) |
| Token count | Prefer 1 token; compound `<primary>-<sub>` to narrow (e.g. `admin-jobs`, `team-digest`) |
| Suffix drop | `email_service` → `email`, `ai_engine` → `ai` (Java/.NET `Service`/`Manager` same) |

| New scope decision | Action |
|--------------------|--------|
| Synonym exists in `.commit-scopes` | Reuse — never coin duplicate (`auth` vs `authentication` vs `login`) |
| Genuinely new concept | Add to `.commit-scopes` in same PR as first commit using it |
| Can't update file now (hotfix, fast flow) | Drop `(<scope>)` (valid CC) or use `--quick`. Update `.commit-scopes` before merge |

> 🚫 Never invent a generic scope (`core`, `misc`) to fill format. No-scope flags "needs categorization"; invented scope masks the gap.

### `.commit-scopes` file

Plain text — one scope per line, `#` lines are comments, blanks/whitespace trimmed.

Sections in this repo's file: top-level modules → sub-feature compounds → cross-cutting. SKILL.md is portable — copy to another repo + create that repo's own `.commit-scopes`, no skill edit needed.

## Examples

**feat with body:**

```
feat(auth): thêm JWT refresh token rotation (WRA-201)

Implement sliding expiration cho refresh token, revoke
token cũ khi phát hiện reuse.
```

**fix one-liner:**

```
fix(billing): tính sai VAT cho đơn hàng có discount (WRA-334)
```

**refactor with body:**

```
refactor(order): tách OrderService thành các handler nhỏ (WRA-412)

Không đổi behavior, chuẩn bị cho việc thêm payment provider.
```

**breaking change:**

```
feat(api)!: đổi response format endpoint /users (WRA-450)

BREAKING CHANGE: field `user_id` đổi thành `id`. Clients
phải cập nhật trước khi deploy.
```

**revert:**

```
revert: feat(auth): thêm JWT refresh token rotation (WRA-501)

This reverts commit 7cd2ed6693da5f5d70751084d20c915c54b9f37d.

Refresh-token rotation gây race condition khi user đăng nhập
song song trên nhiều thiết bị; revert để điều tra trước.

Refs WRA-201
```

| Revert element | Rule |
|----------------|------|
| Subject | Take original header, **replace** `(JIRA-original)` with `(JIRA-revert-task)` |
| Invariant preserved | Subject still ends with exactly one `(<JIRA-ID>)` |
| Original commit identity | SHA in `This reverts commit <full-SHA>.` line (auto-generated by `git revert`) |
| Original ticket trace | `Refs <JIRA-original>` footer (optional) |
| Why-explanation | In body, before footer |

## Quick mode

**Trigger**: `--quick` in args (e.g. `/commit WRA-9 --quick`).

| Aspect | Rule |
|--------|------|
| Format | `<type>: <subject> (<JIRA-ID>)` — no scope, ever |
| Body | Skipped, even when meaningful |
| Header length | ≤ 72 chars (tighter than normal mode) |
| Mandatory | `type`, Jira ID, imperative subject, no period |

**Use for**: hotfix · dep bump · typo fix · internal tool · small chore
**Don't use for**: `feat`/`refactor` needing why-body · breaking change · multi-module change (drop `--quick`, use no-scope normal commit)

### Examples

```
fix: timeout retry khi gọi Jira chậm (WRA-501)
```

```
build: bump litellm 1.50 → 1.52 (WRA-502)
```

```
docs: thêm troubleshooting cho IMAP TLS (WRA-503)
```

> Dep bump → `build` (build system / packaging) per CC spec, not `chore`. `chore` only for housekeeping fitting no other type.

## WIP / Spike

| Element | Rule |
|---------|------|
| Type | `chore` (always) |
| Keyword | `wip` or `spike` — lowercase, first word of subject |
| Scope | None |
| Format | `chore: <wip\|spike> <desc> (<JIRA-ID>)` |

```
chore: wip refactor luồng auth (WRA-123)
```

```
chore: spike test kết nối Redis (WRA-999)
```

| Lifecycle | Rule |
|-----------|------|
| WIP → main | **Must** squash/rebase before merge. Main never holds raw `wip` chain |
| Spike → main | Keep if documents a decision; delete if throwaway — decided in PR review |
| Hotfix | NO — that's a real `fix:` |

> Pairs naturally with `--quick`: `/commit WRA-123 --quick` (no scope, no body, lightweight).
