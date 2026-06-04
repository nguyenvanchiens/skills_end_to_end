---
name: gitlab-sync
description: Resolve conflict khi sync `main → builds/dev/<app>` để deploy QA trong GitLab flow của team. Hỗ trợ monorepo multi-app — mỗi app có nhánh `builds/dev/<app>` riêng (vd `portal-web-admin`, `gift-web-admin`, `gift-api`, `portal-api`...). Skill bắt buộc dùng nhánh trung gian `sync/*` để giữ nguyên tắc code chảy 1 chiều — KHÔNG bao giờ để code `builds/dev/*` chảy ngược vào `main`. Use khi user nói "sync main to dev-<app>", "sync main vào builds/dev", "resolve conflict builds/dev", "merge main vào build conflict", "tạo nhánh sync", "deploy QA bị conflict", "list build branches", "kiểm tra build hygiene", hoặc khi PR `main → builds/dev/*` bị conflict.
---

# GitLab Sync (main → builds/dev/<app>)

Skill này pair với `gitlab-flow`. Trong khi `gitlab-flow` lo phần `feature → main`, skill này lo phần **sync `main → builds/dev/<app>`** khi cần deploy QA mà gặp conflict.

> **Phạm vi**: Chỉ tập trung vào `main → builds/dev/<app>`. Các flow khác (`release → builds/prod`, `cut release`, `cherry-pick hotfix`) trong thực tế gần như không conflict (FF được) hoặc chỉ làm 1 vài lần/quý → không cần automation, Maintainer làm tay được. Nếu sau này phát sinh nhu cầu thì mở rộng skill.

## Nguyên tắc cốt lõi

> **Code chỉ chảy 1 chiều từ `main` → `builds/dev/<app>`.**
> **Conflict resolve TRÊN nhánh trung gian `sync/*`, PR ngược về `builds/dev/<app>` — KHÔNG BAO GIỜ lôi code `builds/dev/*` vào `main`.**

### Hierarchy của branch (upstream → downstream) — Monorepo multi-app

```
              ┌──→ builds/dev/portal-web-admin   ─┐
              ├──→ builds/dev/gift-web-admin       │
main  ────────┼──→ builds/dev/gift-api             ├──→ deploy QA
              ├──→ builds/dev/portal-api           │
              └──→ builds/dev/<other-app>         ─┘
```

| Tier | Branch | Vai trò |
|---|---|---|
| **Source of truth** | `main` | Integration, đã review, đã pass gate. 1 cho cả monorepo |
| **Deploy trigger (QA)** | `builds/dev/<app>` | 1 nhánh cho **mỗi app**. Chỉ là điểm trigger CI/CD, KHÔNG chứa code mới |
| **Ephemeral** | `sync/*` | Tạm thời, xoá ngay sau khi MR merged |

> **Key insight monorepo**: code chung 1 `main` nhưng nhiều `builds/dev/<app>` vì mỗi app deploy độc lập. Mỗi lần sync chỉ động vào **1 app**, không sync hàng loạt trừ khi user yêu cầu rõ.

### Rule không bao giờ vi phạm

1. 🚫 **KHÔNG commit thẳng vào `builds/dev/*`** — kể cả "sửa nhanh 1 dòng"
2. 🚫 **KHÔNG PR `builds/dev/* → main`** — chiều ngược, làm rác main
3. 🚫 **KHÔNG merge `feature/* → builds/dev/*` trực tiếp** — phải qua `main`
4. 🚫 **KHÔNG resolve conflict bằng cách kéo `builds/dev` vào nhánh feature** — code rác sẽ leo lên main
5. ✅ **Conflict `main → builds/dev/<app>` → resolve trên `sync/main-to-dev-<app>`** rồi PR vào `builds/dev/<app>`

## Conventions

### Sync branch naming

Format: `sync/main-to-dev-<app>[-<date>]`

| Use case | Tên nhánh ví dụ |
|---|---|
| Sync `main` → `builds/dev/portal-web-admin` | `sync/main-to-dev-portal-web-admin` |
| Sync `main` → `builds/dev/gift-api` | `sync/main-to-dev-gift-api` |
| Sync `main` → `builds/dev/portal-api` | `sync/main-to-dev-portal-api` |
| Sync 2 lần trong cùng ngày trên cùng app | thêm `-2026-05-12-2` (lần 2 trong ngày) |

| Rule | Detail |
|---|---|
| Prefix | Luôn `sync/` — KHÔNG dùng `feature/`, `chore/`, `fix/` |
| Format | `sync/main-to-dev-<app>` — đọc 1 phát biết chiều và app nào |
| App slug | **Bắt buộc**. Khớp 100% với slug ở `builds/dev/<app>` |
| Date suffix | Optional, thêm khi cần phân biệt nhiều lần sync trong ngày |
| Tuổi thọ | **Ephemeral** — xoá ngay sau khi MR merged |
| Length | Target ≤ 60 chars. App slug dài quá → vẫn giữ nguyên (đừng viết tắt) |

### Commit message khi resolve conflict

Format: `chore(sync): resolve conflict main → builds/dev/<app>`

| Ví dụ |
|---|
| `chore(sync): resolve conflict main → builds/dev/portal-web-admin` |
| `chore(sync): resolve conflict main → builds/dev/gift-api` |

- KHÔNG dùng type `feat`/`fix` — sync không thêm logic mới, chỉ reconcile
- Body (optional): mô tả file conflict + bên nào được giữ
- 🚫 KHÔNG `Co-Authored-By: Claude ...` (kế thừa rule từ `gitlab-flow`)

### Target branch của MR

| Sync branch | MR target | KHÔNG được set target = |
|---|---|---|
| `sync/main-to-dev-<app>` | `builds/dev/<app>` | ❌ `main`, ❌ `builds/dev/<app-khác>` |

> ⚠️ **Self-check trước khi tạo MR**:
> 1. Target **bắt buộc** là `builds/dev/<app>` — KHÔNG được PR vào `main`
> 2. Tên app trong sync branch phải khớp với app trong target. Vd `sync/main-to-dev-gift-api` → target **bắt buộc** là `builds/dev/gift-api`, KHÔNG được PR sang `builds/dev/portal-api`

## Triggers & Procedures

### "list build branches" / "show me các app trong repo" / "có những builds dev nào"

Liệt kê toàn bộ `builds/dev/*` hiện có — giúp user pick đúng app khi sync.

```bash
git fetch --all --prune
git ls-remote --heads origin "refs/heads/builds/dev/*" | awk '{print $2}' | sed 's|refs/heads/builds/dev/||' | sort
```

Output ví dụ:
```
gift-api
gift-web-admin
portal-api
portal-web-admin
```

Format report cho user:

```
## QA apps (builds/dev/<app>)
- gift-api
- gift-web-admin
- portal-api
- portal-web-admin
```

Hỏi tiếp: "Bạn muốn sync app nào?" → dùng trigger `sync main to dev-<app>`.

### "sync main to dev-<app>" / "sync main vào builds/dev/<app>" / "resolve conflict builds/dev/<app>"

Trigger chính. Xảy ra khi cần deploy QA cho 1 app nhưng `main → builds/dev/<app>` không fast-forward được.

**Step 0 — Detect app slug từ input**:

| Input pattern | Hành động |
|---|---|
| `sync main to dev-gift-api` | App = `gift-api`, target = `builds/dev/gift-api` |
| `sync main vào builds/dev/portal-web-admin` | App = `portal-web-admin` |
| `resolve conflict gift-api` | App = `gift-api`, target = `builds/dev/gift-api` |
| `sync main to dev` (không có app) | STOP, gọi trigger `list build branches` để user pick |
| `sync QA` (chung chung) | STOP, hỏi: app nào? |

Verify app tồn tại trước khi tiếp tục:

```bash
git fetch --all --prune
git ls-remote --heads origin "refs/heads/builds/dev/<app>" | head -1
```

- Output rỗng → STOP, báo "Không tìm thấy `builds/dev/<app>`. Chạy `list build branches` xem các app có sẵn"
- Có output → tiếp tục Step 1

**Step 1 — Verify tình huống có cần sync không**:

```bash
git log --oneline origin/main..origin/builds/dev/<app>          # commit có trên target mà main không có
git log --oneline origin/builds/dev/<app>..origin/main          # commit có trên main mà target không có
```

| Output | Ý nghĩa | Hành động |
|---|---|---|
| `origin/builds/dev/<app>..origin/main` rỗng | Main không có gì mới so với target | STOP, báo "không có gì để sync" |
| `origin/main..origin/builds/dev/<app>` rỗng | Target là subset của main → FF được | Không cần sync branch, có thể PR `main → builds/dev/<app>` trực tiếp |
| `origin/main..origin/builds/dev/<app>` có merge commit only | Bình thường (kết quả sync trước) | Tiếp tục Step 2 |
| `origin/main..origin/builds/dev/<app>` có non-merge commit | Vi phạm rule "không commit thẳng builds/*" | Cảnh báo user, đề xuất chạy `kiểm tra build hygiene` trước khi sync |

**Step 2 — Tạo sync branch từ `main`** (KHÔNG từ target):

```bash
git checkout main
git pull origin main
git checkout -b sync/main-to-dev-<app>
```

Vd cho `portal-web-admin`: `git checkout -b sync/main-to-dev-portal-web-admin`

**Step 3 — Merge target VÀO sync branch để lộ conflict**:

```bash
git merge origin/builds/dev/<app>
```

| Kết quả | Hành động |
|---|---|
| `Already up to date` | Không có gì để resolve — báo user, xoá sync branch, kết thúc |
| `Fast-forward` (rare) | Không có conflict thật — có thể FF trực tiếp ở Step 6 |
| `CONFLICT (content): ...` | Bình thường, sang Step 4 |
| `error: refusing to merge unrelated histories` | Branch lệch nghiêm trọng — STOP, hỏi user |

**Step 4 — Resolve conflict (giữ phía main)**:

```bash
git status                                # list file conflict
git diff --name-only --diff-filter=U      # chỉ list file đang conflict
```

Quy tắc resolve:

| Tình huống | Bên nên giữ | Lý do |
|---|---|---|
| Code logic (function, component, API) | **`main`** (HEAD) | Main là source of truth, đã review |
| Config QA-specific trên `builds/dev` (API URL QA, debug flag) | **target** (incoming) | QA env config không được leak lên main |
| Resolution không rõ | **STOP, hỏi user** | Không tự đoán — sync sai sẽ ảnh hưởng deploy |

**Cách giữ phía main nhanh** (khi chắc chắn 100%):

```bash
git checkout --ours <file>       # giữ phía HEAD = main (vì sync branch tạo từ main)
git add <file>
```

**Cách giữ phía target**:

```bash
git checkout --theirs <file>
git add <file>
```

**Cách edit thủ công** (default — nên dùng cho code logic):
- Mở file, xoá marker `<<<<<<<`, `=======`, `>>>>>>>`
- Giữ đúng phiên bản logic mong muốn
- `git add <file>`

**Step 5 — Verify + commit**:

```bash
git status                       # phải sạch, không còn file `U`
git diff --cached --stat         # review những file thay đổi
```

Commit message:

```bash
git commit -m "$(cat <<'EOF'
chore(sync): resolve conflict main → builds/dev/<app>

Conflict file: <list>
Resolution: giữ phiên bản main cho logic, giữ phía target cho config env.
EOF
)"
```

> 🚫 KHÔNG `Co-Authored-By: Claude` (xem `gitlab-flow` rule).

**Step 6 — Push gate**:

1. Báo commit hash + nội dung
2. **DỪNG, hỏi user**: "Đã commit `<hash>` resolve conflict. Push lên remote và tạo MR không?"
3. Đợi xác nhận:

```bash
git push -u origin sync/main-to-dev-<app>
```

**Step 7 — Tạo MR `sync/* → builds/dev/<app>`** (KHÔNG về main):

```bash
glab mr create \
  --source-branch sync/main-to-dev-<app> \
  --target-branch builds/dev/<app> \
  --title "chore(sync): main → builds/dev/<app>" \
  --description "$(cat <<'EOF'
## Summary
- Sync code từ main về builds/dev/<app> để deploy QA
- Resolve conflict trên các file: <list>

## Resolution
- Logic code: giữ phía main
- Config env (nếu có): giữ phía target

## Test plan
- [ ] CI build pass trên sync branch
- [ ] Sau khi merge, deploy QA app <app> chạy được
- [ ] Smoke test golden path của app <app>
EOF
)" \
  --remove-source-branch
```

**Self-check trước khi chạy lệnh trên**:
- [ ] `--source-branch` bắt đầu bằng `sync/`
- [ ] `--target-branch` là `builds/dev/<app>` (KHÔNG phải `main`)
- [ ] App slug trong source khớp app slug trong target (`sync/main-to-dev-gift-api` → `builds/dev/gift-api`, không phải `builds/dev/portal-api`)
- [ ] Title có prefix `chore(sync):`
- [ ] `--remove-source-branch` có mặt (sync branch là ephemeral)

**Step 8 — Sau khi MR merged**:

```bash
git checkout main
git fetch --prune
git branch -d sync/main-to-dev-<app>      # local đã xoá
# Remote tự xoá bởi --remove-source-branch
```

Báo user: deploy QA cho app `<app>` giờ có thể trigger (Maintainer push tag / chạy pipeline).

### "sync main to dev-all" / "sync main vào tất cả builds/dev" / "deploy QA tất cả apps"

Khi cần sync `main` sang **nhiều** `builds/dev/<app>` cùng lúc. Skill chạy tuần tự **từng app** — KHÔNG gộp 1 sync branch chung.

**Step 1 — List + confirm**:

1. Chạy trigger `list build branches` để lấy danh sách
2. Hiển thị danh sách cho user, hỏi: "Sync hết tất cả app hay chỉ chọn vài app? Liệt kê app cần sync (cách nhau dấu phẩy) hoặc 'all'"
3. Đợi user pick

**Step 2 — Loop qua từng app**:

Với mỗi app trong danh sách user chọn:

1. Verify như Step 1 của trigger "sync main to dev-<app>"
2. Nếu **không cần sync** (target = subset của main) → skip, báo `[<app>] đã đồng bộ, skip`
3. Nếu **cần sync** → chạy đầy đủ Step 2-8 cho app đó
4. **DỪNG sau mỗi app**, hỏi user: "App `<app>` đã có MR, xác nhận tiếp app kế tiếp không?"
5. User: "skip"/"next" → bỏ qua; "stop" → dừng toàn bộ

**Step 3 — Tóm tắt cuối**:

```
## Sync summary
✅ Đã tạo MR (chờ merge):
  - sync/main-to-dev-portal-web-admin → !123
  - sync/main-to-dev-gift-api → !124

⏭ Skip (đã đồng bộ):
  - portal-api
  - gift-web-admin

❌ Lỗi:
  - notification-service (lý do: ...)
```

> ⚠️ KHÔNG dùng 1 sync branch chung cho nhiều `builds/dev/<app>`. Mỗi app có config khác nhau, conflict khác nhau, MR phải tách riêng để dev đúng app review được.

### "kiểm tra build hygiene" / "check builds/dev/<app> sạch không" / "audit all dev builds"

Audit định kỳ: phát hiện vi phạm rule "không commit thẳng `builds/dev/*`". Có 2 mode: **single app** hoặc **toàn bộ**.

**Step 1 — Xác định scope**:

| User input | Mode |
|---|---|
| `kiểm tra build hygiene portal-web-admin` | Single app |
| `audit all dev builds` | All apps — auto-list rồi loop |

**Step 2 — So sánh từng build với main**:

```bash
git fetch --all --prune
git log --oneline origin/main..origin/builds/dev/<app>     # commit có trên build mà main không có
```

| Output | Diễn giải |
|---|---|
| Rỗng | ✅ Sạch — `builds/dev/<app>` đúng là subset của main |
| Có merge commit only (`Merge branch 'main'`) | ⚠️ Bình thường — kết quả các lần sync trước. Acceptable |
| Có commit không phải merge | ❌ **VI PHẠM** — có người commit thẳng vào `builds/dev/*` |

**Step 3 — Nếu vi phạm, list ra**:

```bash
git log origin/main..origin/builds/dev/<app> --no-merges --pretty=format:"%h %an %ad %s" --date=short
```

Báo user dạng table:

```
## ❌ builds/dev/portal-web-admin — 3 commit vi phạm
| Hash | Author | Date | Subject |
|---|---|---|---|
| abc1234 | nguyenvana | 2026-05-10 | hotfix nhanh login |
| def5678 | tranthib  | 2026-05-08 | sửa config QA |
| 9876543 | lehvc     | 2026-05-05 | test feature flag |
```

**Step 4 — Đề xuất cleanup cho từng commit**:

Option A — Cherry-pick về main (nếu commit có giá trị):
- Tạo `feature/<TASK-ID>-recover-from-build-<app>` nếu thực sự cần
- Cherry-pick từng commit
- PR về main như feature thường

Option B — Reset build về main (nếu commit là rác):
- Cần Maintainer + force-push permission
- Cảnh báo: sẽ mất commit đó vĩnh viễn
- Hỏi user xác nhận 2 lần trước khi đề xuất lệnh `git push --force-with-lease`

> KHÔNG tự chạy force-push. Đề xuất lệnh để user/Maintainer chạy.

**Step 5 — Báo cáo tổng** (Mode all):

```
## Audit summary
✅ Sạch (3): gift-api, portal-api, gift-web-admin
❌ Vi phạm (1):
  - builds/dev/portal-web-admin (3 commit lạ)
```

## Quick decision: user nói gì → trigger gì

| User nói | Trigger |
|---|---|
| "có app nào", "list builds", "show me apps" | `list build branches` |
| "sync main về QA cho portal-web-admin" | `sync main to dev-portal-web-admin` |
| "sync main về dev của gift-api" | `sync main to dev-gift-api` |
| "deploy QA hết các app" | `sync main to dev-all` |
| "check build/dev có sạch không" | `kiểm tra build hygiene <app>` |
| "audit all builds" | `kiểm tra build hygiene` (Mode all) |

## Out of scope (làm tay nếu cần)

Các flow sau **KHÔNG** trong scope skill — Maintainer làm tay vì:
- Tần suất thấp (1 lần/quý hoặc ít hơn)
- Thực tế hiếm conflict (FF được)
- Cần judgment cao, không nên automation

| Flow | Tần suất | Lý do skip |
|---|---|---|
| Cut release branch `main → release/v<X.Y>` | Mỗi major/minor release | Không có conflict (release chưa tồn tại) |
| Deploy prod `release/v<X.Y> → builds/prod/<app>` | Release day | Gần như luôn FF (release = snapshot, builds/prod chỉ theo đuôi) |
| Cherry-pick hotfix về main | Khi có hotfix | Hiếm, Maintainer judgement |
| Cherry-pick bugfix release về main | Cuối release cycle | Hiếm, Maintainer judgement |

Nếu sau này 1 trong các flow trên trở nên thường xuyên + hay conflict → mở rộng skill.

## Safety rules

- 🚫 **KHÔNG tạo MR `builds/dev/* → main`** dưới bất kỳ hình thức nào, kể cả qua nhánh trung gian. Nếu user yêu cầu, từ chối và giải thích vi phạm rule 1-chiều
- 🚫 **KHÔNG `git push --force`** vào `main`, `builds/dev/*` — kể cả `--force-with-lease`. Reset chỉ làm khi user/Maintainer chủ động ra lệnh
- 🚫 **KHÔNG `git merge --strategy=ours`** để "fake resolve" — sẽ skip thay đổi thật, gây lệch code thầm lặng
- 🚫 **KHÔNG dùng `git checkout --ours <file>` cho code logic** mà không đọc qua diff — chỉ dùng khi đã xác nhận phía main đúng
- ✅ **Mọi sync branch là ephemeral** — đảm bảo `--remove-source-branch` khi tạo MR
- ✅ **Hỏi user trước khi resolve** nếu không chắc bên nào đúng — đặc biệt với file config, .env, route definition
- ✅ **Verify target branch của MR** trước khi `glab mr create` — đọc lại `--target-branch` 2 lần
- ✅ **Sau mỗi sync MR merged**: xoá local sync branch + `git fetch --prune` để dọn reference cũ

## Tools required

- `git` (luôn có)
- `glab` (GitLab CLI) — cần để tạo MR sync. Tham khảo install: https://gitlab.com/gitlab-org/cli

## Interaction với gitlab-flow

| Tình huống | Dùng skill |
|---|---|
| Feature mới từ Jira task → main | `gitlab-flow` |
| Review MR feature → main | `gitlab-flow` |
| Merge feature MR vào main | `gitlab-flow` |
| Sau khi feature merged main, cần deploy QA | `gitlab-sync` (main → builds/dev/<app>) |
| Kiểm tra builds/dev có sạch không | `gitlab-sync` (kiểm tra build hygiene) |

2 skill này độc lập nhưng bổ sung nhau — `gitlab-flow` lo "code đi vào main", `gitlab-sync` lo "main đi ra `builds/dev/<app>` để deploy QA".
