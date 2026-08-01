# Commit reference

Bảng tra cứu cho trigger `commit and push` của `gitlab-flow`. **Đọc file này khi phân vân**, không cần đọc mỗi lần commit — format và các bước bắt buộc đã nằm inline trong `SKILL.md`.

Mở file này khi: không chắc chọn `type` nào · cần footer `Closes`/`Refs`/`BREAKING CHANGE` · phân vân đặt `scope` · làm commit WIP/Spike · làm commit revert.

## Allowed types

| Type | Ý nghĩa | Version bump |
|---|---|---|
| `feat` | Tính năng mới | MINOR (1.X.0) |
| `fix` | Sửa bug | PATCH (1.0.X) |
| `perf` | Cải thiện performance | PATCH |
| `refactor` | Refactor không đổi behavior | — |
| `docs` | Tài liệu | — |
| `test` | Thêm/sửa test | — |
| `build` | Build system / dependency / packaging | — |
| `style` | Format code (whitespace, lint) | — |
| `chore` | Maintenance, không fit type khác | — |
| `ci` | CI/CD config | — |
| `revert` | Revert commit cũ | — |

> Breaking change là *modifier*, không phải type riêng. Suffix `!` hoặc footer `BREAKING CHANGE:` → MAJOR bump (X.0.0).

## Footer

Vị trí: sau body, ngăn bằng dòng trắng. Format: `Token: value` (CC) hoặc `Token #issue` (GitHub-style).

| Footer | Khi dùng |
|---|---|
| `BREAKING CHANGE: <desc>` | Bắt buộc khi header có `!`. Mô tả impact + migration |
| `Closes <TASK-ID>` | Trigger Jira-GitLab auto-close khi merge. Skip nếu đã auto-close từ subject mention (kiểm tra 1-2 ticket merged gần đây để confirm) — tránh trigger 2 lần |
| `Refs <TASK-ID>` | Reference Jira khác (related nhưng không close) |
| `Co-authored-by: Name <email>` | Real pair-programming. KHÔNG auto-insert AI |
| `Reviewed-by: Name <email>` | Optional — chỉ nếu team convention |

| Rule | Áp dụng |
|---|---|
| Đừng lặp Task ID | Đã có trong subject `(<TASK-ID>)` rồi → bỏ ở footer trừ khi cần keyword `Closes`/`Refs` |
| Token case | PascalCase hoặc kebab-case (`Reviewed-by`, `Co-authored-by`); `BREAKING CHANGE` uppercase per spec |

## Scope

**Lookup**: đọc `.commit-scopes` ở repo root → fallback `git log --pretty=format:%s | grep -oE '\([^)]+\):' | sort -u`.

| Convention | Detail |
|---|---|
| Case | lowercase, kebab-case (`-`, không `_`) |
| Token count | Prefer 1 token; compound `<primary>-<sub>` để narrow (vd `admin-jobs`, `team-digest`) |
| Suffix drop | `email_service` → `email`, `ai_engine` → `ai`, Java/.NET `Service`/`Manager` tương tự |

| Quyết định new scope | Hành động |
|---|---|
| Synonym đã có trong `.commit-scopes` | Reuse — đừng coin trùng (`auth` vs `authentication` vs `login`) |
| Genuinely new concept | Add vào `.commit-scopes` cùng PR với commit đầu tiên dùng nó |
| Không update file được lúc đó (hotfix, fast flow) | Drop `(<scope>)` (valid CC) hoặc dùng `--quick`. Update `.commit-scopes` trước khi merge |

> 🚫 Đừng invent generic scope (`core`, `misc`) để fill format. No-scope flags "needs categorization"; invented scope masks the gap.

**`.commit-scopes` file**: plain text — 1 scope/dòng, dòng `#` là comment, blank/whitespace trimmed.

## Quick mode

**Trigger**: thêm `--quick` ("commit and push --quick" / "quick commit").

| Aspect | Rule |
|---|---|
| Format | `<type>: <subject> (<TASK-ID>)` — không scope, ever |
| Body | Skip, kể cả meaningful |
| Header length | ≤72 chars (chặt hơn normal) |
| Mandatory | `type`, TASK-ID, imperative subject, không chấm cuối |

**Use for**: hotfix · dep bump · typo fix · internal tool · small chore
**Don't use for**: `feat`/`refactor` cần why-body · breaking change · multi-module change (drop `--quick`, dùng no-scope normal)

> Dep bump → `build` (build system / packaging) per CC spec, không `chore`. `chore` chỉ cho housekeeping không fit type khác.

## WIP / Spike

| Element | Rule |
|---|---|
| Type | `chore` (always) |
| Keyword | `wip` hoặc `spike` — lowercase, từ đầu của subject |
| Scope | None |
| Format | `chore: <wip\|spike> <desc> (<TASK-ID>)` |

```
chore: wip refactor luồng auth (WRA-123)
chore: spike test kết nối Redis (WRA-999)
```

| Lifecycle | Rule |
|---|---|
| WIP → main | **Bắt buộc** squash/rebase trước merge. Main không bao giờ giữ chuỗi `wip` raw |
| Spike → main | Giữ nếu document được decision; xóa nếu throwaway — quyết trong PR review |
| Hotfix | KHÔNG — đó là `fix:` thật |

> Pair tự nhiên với `--quick`: "commit and push --quick" (no scope, no body, lightweight).

## Examples

**feat with body**:
```
feat(auth): thêm JWT refresh token rotation (WRA-201)

Implement sliding expiration cho refresh token, revoke
token cũ khi phát hiện reuse.
```

**fix one-liner**:
```
fix(billing): tính sai VAT cho đơn hàng có discount (WRA-334)
```

**refactor with body**:
```
refactor(order): tách OrderService thành các handler nhỏ (WRA-412)

Không đổi behavior, chuẩn bị cho việc thêm payment provider.
```

**breaking change**:
```
feat(api)!: đổi response format endpoint /users (WRA-450)

BREAKING CHANGE: field `user_id` đổi thành `id`. Clients
phải cập nhật trước khi deploy.
```

**revert**:
```
revert: feat(auth): thêm JWT refresh token rotation (WRA-501)

This reverts commit 7cd2ed6693da5f5d70751084d20c915c54b9f37d.

Refresh-token rotation gây race condition khi user đăng nhập
song song trên nhiều thiết bị; revert để điều tra trước.

Refs WRA-201
```

| Revert element | Rule |
|---|---|
| Subject | Lấy original header, **replace** `(JIRA-original)` bằng `(JIRA-revert-task)` |
| Invariant preserved | Subject vẫn kết thúc với exactly 1 `(<TASK-ID>)` |
| Original commit identity | SHA trong dòng `This reverts commit <full-SHA>.` (auto-generated bởi `git revert`) |
| Original ticket trace | `Refs <JIRA-original>` footer (optional) |
| Why-explanation | Trong body, trước footer |
