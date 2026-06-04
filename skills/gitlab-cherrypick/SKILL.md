---
name: gitlab-cherrypick
description: Cherry-pick commit từ `main` vào `release/<app>/<version>` của monorepo multi-app để cut patch release / backport fix. Workflow interactive — user khai báo số ngày lùi (vd "5 ngày"), skill list commit history main (đã loại trừ commit có sẵn trên release), user pick number các commit cần backport. Skill bắt buộc dùng nhánh trung gian `cherry/*` để giữ nguyên tắc code chảy 1 chiều — KHÔNG commit thẳng vào `release/<app>/<version>`. Use khi user nói "cherry-pick to release/<app>/<v>", "cherry-pick main vào release", "backport WRA-40 sang release", "đưa commit vào release", "list commit main 5 ngày", "list releases", "patch release", hoặc khi cần đưa fix/feature từ main sang release branch.
---

# GitLab Cherry-pick (main → release/<app>/<version>)

Skill này pair với `gitlab-flow` và `gitlab-sync`. Trong khi:
- `gitlab-flow` lo `feature → main`
- `gitlab-sync` lo `main → builds/dev/<app>` (deploy QA)

Skill này lo phần **`main → release/<app>/<version>`** — backport commit cụ thể để cut patch release.

> **Phạm vi**: Chỉ tập trung `main → release/<app>/<version>`. Cut release branch mới (`release/<app>/<v-mới>`), deploy prod, tag release — out of scope, Maintainer làm tay.

## Nguyên tắc cốt lõi

> **Code chỉ chảy 1 chiều từ `main` → `release/<app>/<version>`.**
> **Cherry-pick TRÊN nhánh trung gian `cherry/*`, PR ngược về `release/<app>/<version>` — KHÔNG BAO GIỜ lôi code `release/*` vào `main`.**

### Hierarchy của branch — Monorepo multi-app

```
              ┌──→ release/portal-web-admin/v1.2.0
              ├──→ release/portal-web-admin/v1.3.0
main  ────────┼──→ release/gift-api/v0.5.3                  ──→ deploy prod (Maintainer làm tay)
              ├──→ release/gift-api/v0.6.0
              └──→ release/<app>/<other-version>
```

| Tier | Branch | Vai trò |
|---|---|---|
| **Source of truth** | `main` | Integration. Mọi feature/fix đều land ở đây trước |
| **Release snapshot** | `release/<app>/<version>` | 1 nhánh cho **mỗi app + mỗi version**. Frozen sau khi cut, chỉ nhận cherry-pick patch |
| **Ephemeral** | `cherry/*` | Tạm thời, xoá ngay sau khi MR merged |

> **Key insight**: cherry-pick không phải full sync (như `gitlab-sync`) — chỉ đưa **subset commit** từ main về release. Lý do: release đã frozen, chỉ nhận patch có chủ ý, không nhận hết mọi thay đổi của main.

### Rule không bao giờ vi phạm

1. 🚫 **KHÔNG commit thẳng vào `release/<app>/<version>`** — kể cả "fix nhanh 1 dòng"
2. 🚫 **KHÔNG PR `release/<app>/* → main`** — chiều ngược, làm rác main
3. 🚫 **KHÔNG cherry-pick commit chưa land main** — main là source of truth, cherry phải có gốc rõ ràng
4. 🚫 **KHÔNG dùng `git merge main` để "đồng bộ" release** — sẽ kéo HẾT main về, vỡ frozen state. Phải cherry-pick từng commit có chủ ý
5. ✅ **Mọi cherry-pick → tạo MR `cherry/* → release/<app>/<version>`** để team review trước khi vào release

## Conventions

### Cherry branch naming

Format: `cherry/main-to-release-<app>-<version>[-<desc>]`

| Use case | Tên nhánh ví dụ |
|---|---|
| Cherry-pick 1-2 commit vào `release/portal-web-admin/v1.2.0` | `cherry/main-to-release-portal-web-admin-v1.2.0` |
| Cherry-pick nhiều commit cho 1 fix cụ thể | `cherry/main-to-release-gift-api-v0.5.3-WRA-334-vat-fix` |
| Cherry-pick 2 lần trong ngày trên cùng release | thêm `-2026-05-20-2` |

| Rule | Detail |
|---|---|
| Prefix | Luôn `cherry/` — KHÔNG dùng `feature/`, `fix/`, `sync/` |
| Format cơ bản | `cherry/main-to-release-<app>-<version>` — đọc 1 phát biết chiều, app, version |
| Slash → dash | `release/<app>/<v>` (slash) → trong cherry branch dùng dash: `<app>-<v>` |
| Optional suffix | Thêm `-<TASK-ID>-<short>` khi cherry-pick có theme rõ (vd hotfix VAT) |
| Tuổi thọ | **Ephemeral** — xoá ngay sau khi MR merged |
| Length | Target ≤ 70 chars. Quá dài → bỏ suffix `-<desc>` |

### Cherry-pick commit message

Mặc định dùng `git cherry-pick -x <SHA>` — git tự thêm dòng `(cherry picked from commit <SHA>)` vào cuối body. **Giữ nguyên** dòng này để trace về commit gốc.

| Element | Rule |
|---|---|
| Subject | Giữ NGUYÊN từ commit gốc (KHÔNG sửa) |
| Body | Giữ nguyên + dòng `(cherry picked from commit <SHA>)` cuối |
| TASK-ID | Giữ nguyên `(<TASK-ID>)` cuối subject |
| 🚫 KHÔNG thêm | `Co-Authored-By: Claude ...` (kế thừa rule `gitlab-flow`) |

**Ví dụ commit sau cherry-pick**:
```
fix(billing): tính sai VAT cho đơn hàng có discount (WRA-334)

Tách hàm tính VAT khỏi total để tránh double-apply discount.

(cherry picked from commit def5678abc1234)
```

### Target branch của MR

| Cherry branch | MR target | KHÔNG được set target = |
|---|---|---|
| `cherry/main-to-release-<app>-<v>` | `release/<app>/<v>` | ❌ `main`, ❌ `release/<app>/<v-khác>`, ❌ `builds/dev/*` |

> ⚠️ **Self-check trước khi tạo MR**:
> 1. Target **bắt buộc** là `release/<app>/<version>` — KHÔNG được PR vào `main`
> 2. Version trong cherry branch phải khớp version trong target (`cherry/...-v1.2.0` → `release/<app>/v1.2.0`)

## Triggers & Procedures

### "list releases" / "show me các release branch" / "có release nào"

Liệt kê toàn bộ `release/<app>/<version>` hiện có — giúp user pick đúng target khi cherry-pick.

```bash
git fetch --all --prune
git ls-remote --heads origin "refs/heads/release/*" | awk '{print $2}' | sed 's|refs/heads/||' | sort
```

Output ví dụ:
```
release/gift-api/v0.5.3
release/gift-api/v0.6.0
release/portal-web-admin/v1.2.0
release/portal-web-admin/v1.3.0
```

Format report cho user (group theo app):

```
## Release branches

### portal-web-admin
- v1.2.0
- v1.3.0  (latest)

### gift-api
- v0.5.3
- v0.6.0  (latest)
```

Hỏi tiếp: "Bạn muốn cherry-pick vào release nào?" → trigger `cherry-pick to release/<app>/<v>`.

### "list commit main last <N> days" / "liệt kê commit main 5 ngày" / "show me commit gần đây"

Liệt kê commit trên `main` trong N ngày qua — để user khảo sát trước khi quyết định pick.

**Step 1 — Detect N từ input**:

| Input | N (ngày) |
|---|---|
| `list commit main last 5 days` | 5 |
| `liệt kê commit main 7 ngày` | 7 |
| `show commit main 3 ngày qua` | 3 |
| Không có số | STOP, hỏi: "Lùi bao nhiêu ngày? (default 7)" |

**Step 2 — Fetch + list**:

```bash
git fetch origin main
git log origin/main --since="<N> days ago" --pretty=format:"%h|%an|%ad|%s" --date=short
```

**Step 3 — Format đẹp cho user**:

```
## Commit trên main (5 ngày qua, mới nhất ở dưới)

| # | SHA | Author | Date | Subject |
|---|-----|--------|------|---------|
| 1 | abc1234 | nguyenvana | 2026-05-16 | feat(auth): restrict login domain (WRA-40) |
| 2 | def5678 | tranthib   | 2026-05-17 | fix(billing): tính sai VAT (WRA-334) |
| 3 | 9876543 | lehvc      | 2026-05-18 | refactor(order): tách handler (WRA-412) |
| 4 | 1234abc | nguyenvana | 2026-05-20 | feat(gift): báo cáo POD theo miền (HNCW-317) |
```

> Trigger này độc lập, không yêu cầu release target. Dùng để khảo sát. Nếu user muốn pick → chạy `cherry-pick to release/<app>/<v>`.

### "cherry-pick to release/<app>/<v>" / "cherry-pick main vào release/<app>/<v>" / "backport sang release/<app>/<v>"

Trigger chính. Interactive flow: list commit → user pick → execute.

**Step 0 — Detect release target từ input**:

| Input pattern | Hành động |
|---|---|
| `cherry-pick to release/gift-api/v0.5.3` | App = `gift-api`, version = `v0.5.3` |
| `cherry-pick main vào release/portal-web-admin/v1.2.0` | App = `portal-web-admin`, version = `v1.2.0` |
| `backport sang release/<app>/<v>` | Tương tự, parse app + version |
| `cherry-pick to release` (không có app/version) | STOP, gọi `list releases` để user pick |
| `cherry-pick WRA-40` (không có target) | STOP, hỏi: "Cherry-pick về release nào? Chạy `list releases` để xem options" |

Verify release branch tồn tại trên remote:

```bash
git fetch --all --prune
git ls-remote --heads origin "refs/heads/release/<app>/<version>" | head -1
```

- Output rỗng → STOP, báo "Không tìm thấy `release/<app>/<version>`. Chạy `list releases` xem các release có sẵn"
- Có output → tiếp tục Step 1

**Step 1 — Hỏi số ngày lùi**:

Hỏi user: "Lùi bao nhiêu ngày để list commit main? (default 7)"

| User trả lời | N |
|---|---|
| `5` / `5 ngày` / `5d` | 5 |
| `all` / `tất cả` | KHÔNG since filter (list từ phân kỳ với release) |
| Enter / `default` | 7 |

> Nếu user gõ trigger kèm `last <N> days` ngay từ đầu → skip hỏi, dùng N đó luôn.

**Step 2 — List commit main, loại trừ commit có sẵn trên release**:

```bash
# Commit có trên main nhưng KHÔNG có trên release target
git log origin/main ^origin/release/<app>/<version> \
    --since="<N> days ago" \
    --pretty=format:"%h|%an|%ad|%s" --date=short
```

| Output | Hành động |
|---|---|
| Rỗng | STOP, báo "Không có commit mới trên main trong <N> ngày qua mà chưa có trên release. Thử tăng N hoặc check release đã up-to-date" |
| Có list | Format đẹp như Step 3 của trigger `list commit main` (kèm số `#` để user pick) |

**Step 3 — Hỏi user pick commit**:

Hỏi: "Pick commit cần cherry-pick (theo số `#`):"

| Format user trả lời | Diễn giải |
|---|---|
| `2, 4` | Pick commit số 2 và 4 |
| `1-3` | Pick commit số 1, 2, 3 |
| `2, 4-6, 8` | Mix — pick 2, 4, 5, 6, 8 |
| `all` | Pick toàn bộ list |
| `abc1234, def5678` | Pick bằng SHA (nếu user thích cách này) |
| `cancel` / `huỷ` | Abort, không cherry-pick |

Sau khi user pick, **echo lại danh sách commit sẽ cherry-pick** + hỏi xác nhận:

```
## Sẽ cherry-pick 3 commit sau vào release/gift-api/v0.5.3:

1. def5678 — fix(billing): tính sai VAT (WRA-334)
2. 9876543 — refactor(order): tách handler (WRA-412)
3. 1234abc — feat(gift): báo cáo POD theo miền (HNCW-317)

Xác nhận? (yes / no / edit)
```

| User trả lời | Hành động |
|---|---|
| `yes` / `ok` / `xác nhận` | Sang Step 4 |
| `no` / `huỷ` | Abort |
| `edit` / sửa pick | Quay lại Step 3 |

**Step 4 — Tạo cherry branch từ release target**:

> ⚠️ Cherry branch phải tạo từ `release/<app>/<version>`, **KHÔNG** từ `main`. Vì mục tiêu là apply commit LÊN release state, không phải merge release vào main.

```bash
git fetch origin release/<app>/<version>
git checkout -b cherry/main-to-release-<app>-<version> origin/release/<app>/<version>
```

Vd cho `release/gift-api/v0.5.3`:
```bash
git checkout -b cherry/main-to-release-gift-api-v0.5.3 origin/release/gift-api/v0.5.3
```

**Step 5 — Cherry-pick từng commit theo thứ tự chronological (cũ → mới)**:

> Quan trọng: cherry-pick theo thứ tự **commit date ASC** (cũ nhất trước). Lý do: commit sau có thể depend on commit trước; pick ngược thứ tự sẽ tăng nguy cơ conflict.

```bash
git cherry-pick -x <SHA1> <SHA2> <SHA3>
```

Flag `-x`: thêm dòng `(cherry picked from commit <SHA>)` vào body — bắt buộc để trace.

| Kết quả | Hành động |
|---|---|
| All cherry-pick thành công | Sang Step 7 |
| `CONFLICT (content): ...` ở commit N | Sang Step 6 (resolve), sau đó `git cherry-pick --continue` |
| User muốn skip 1 commit giữa flow | `git cherry-pick --skip` (commit đó bị bỏ, tiếp commit sau) |
| User muốn abort toàn bộ | `git cherry-pick --abort` → quay về state trước Step 5 |

**Step 6 — Resolve conflict (nếu có)**:

```bash
git status                                # list file conflict
git diff --name-only --diff-filter=U      # chỉ list file đang conflict
```

Quy tắc resolve:

| Tình huống | Bên nên giữ | Lý do |
|---|---|---|
| Code logic của feature đang pick | **phía cherry** (incoming) | Đó chính là thay đổi muốn backport |
| Code đã thay đổi trên release (bug fix release-specific) | **release** (HEAD) | Release có path riêng cho fix đó, không được ghi đè |
| File config version-specific (vd version string, changelog) | **release** (HEAD) | Version branch giữ version riêng |
| Resolution không rõ | **STOP, hỏi user** | Cherry-pick sai = release ship code lỗi |

**Cách giữ phía cherry (incoming commit)**:
```bash
git checkout --theirs <file>
git add <file>
```

**Cách giữ phía release (HEAD)**:
```bash
git checkout --ours <file>
git add <file>
```

**Cách edit thủ công** (default — nên dùng cho code logic):
- Mở file, xoá marker `<<<<<<<`, `=======`, `>>>>>>>`
- Merge logic của 2 phía nếu cả hai cần thiết
- `git add <file>`

Sau khi resolve hết:
```bash
git status                       # phải sạch, không còn file `U`
git cherry-pick --continue       # tiếp tục cherry-pick các commit còn lại
```

> Editor sẽ mở để confirm commit message — **giữ nguyên** message gốc + dòng `(cherry picked from commit <SHA>)`. Đừng tự sửa.

**Step 7 — Verify + push gate**:

```bash
git log origin/release/<app>/<version>..HEAD --oneline    # list commit vừa cherry-pick
```

1. Báo user: list các commit mới + tóm tắt
2. **DỪNG, hỏi user**: "Đã cherry-pick `<N>` commit vào nhánh `cherry/...`. Push lên remote và tạo MR không?"
3. Đợi xác nhận:

```bash
git push -u origin cherry/main-to-release-<app>-<version>
```

**Step 8 — Tạo MR `cherry/* → release/<app>/<version>`** (KHÔNG về main):

```bash
glab mr create \
  --source-branch cherry/main-to-release-<app>-<version> \
  --target-branch release/<app>/<version> \
  --title "chore(cherry): backport <N> commit từ main → release/<app>/<version>" \
  --description "$(cat <<'EOF'
## Summary
- Cherry-pick <N> commit từ main vào release/<app>/<version>
- Mục đích: <patch release / hotfix / backport feature>

## Commits backported

| Original SHA | Subject |
|---|---|
| def5678 | fix(billing): tính sai VAT (WRA-334) |
| 9876543 | refactor(order): tách handler (WRA-412) |
| 1234abc | feat(gift): báo cáo POD theo miền (HNCW-317) |

## Conflict resolution (nếu có)
- File X: giữ phía cherry (incoming)
- File Y: edit thủ công, merge logic 2 bên

## Test plan
- [ ] CI build pass trên cherry branch
- [ ] Smoke test golden path của app <app> trên release version <v>
- [ ] Regression test các module liên quan tới commit cherry-pick
EOF
)" \
  --remove-source-branch
```

**Self-check trước khi chạy lệnh trên**:
- [ ] `--source-branch` bắt đầu bằng `cherry/`
- [ ] `--target-branch` là `release/<app>/<version>` (KHÔNG phải `main`, KHÔNG phải `builds/dev/*`)
- [ ] App + version trong source khớp target (`cherry/...-gift-api-v0.5.3` → `release/gift-api/v0.5.3`)
- [ ] Title có prefix `chore(cherry):`
- [ ] `--remove-source-branch` có mặt (cherry branch là ephemeral)
- [ ] Description **KHÔNG** chứa `Claude`, `🤖`, `Generated with`, `claude.com`, `Co-Authored-By:`

**Step 9 — Sau khi MR merged**:

```bash
git checkout main
git fetch --prune
git branch -d cherry/main-to-release-<app>-<version>     # local đã xoá
# Remote tự xoá bởi --remove-source-branch
```

Báo user: release `<app>/<v>` giờ có thể được Maintainer tag + deploy prod.

## Quick decision: user nói gì → trigger gì

| User nói | Trigger |
|---|---|
| "có release nào", "list release branches" | `list releases` |
| "list commit main 5 ngày" | `list commit main last 5 days` |
| "cherry-pick fix VAT vào release gift-api v0.5.3" | `cherry-pick to release/gift-api/v0.5.3` (sau đó pick commit theo số) |
| "backport WRA-40 sang release portal-web-admin v1.2.0" | `cherry-pick to release/portal-web-admin/v1.2.0` |
| "patch release portal-web-admin v1.2.0" | `cherry-pick to release/portal-web-admin/v1.2.0` |

## Edge cases

### Commit phụ thuộc lẫn nhau

Nếu commit B build trên commit A (vd A thêm helper, B dùng helper) — phải pick **cả A và B**, theo thứ tự A→B. Skill pick theo chronological ASC nên tự nhiên đúng thứ tự, nhưng nếu user pick chỉ B mà bỏ A → cherry-pick sẽ conflict hoặc fail compile.

**Khuyến nghị**: trước khi confirm pick (Step 3), nếu skill detect các commit pick KHÔNG liên tục trên main → warn user: "Đang skip commit `<SHA-giữa>` — có chắc commit pick không depend on nó không?"

### Commit đã cherry-pick rồi (duplicate)

Nếu user vô tình pick lại commit đã có trên release (vd quên check `list releases`):

```bash
git cherry-pick -x <SHA>
# error: The previous cherry-pick is now empty
```

→ Báo user: "Commit `<SHA>` đã có sẵn trên release. Skip bằng `git cherry-pick --skip` hoặc abort."

Để **phòng** trường hợp này, Step 2 đã loại trừ commit có sẵn (`^origin/release/<app>/<v>`).

### Revert commit trên main → có cần cherry-pick revert?

Nếu main đã revert commit X (vd commit Y = revert of X), nhưng X chưa từng land release → **không cần pick cả 2**, vì release chưa có X. Skill detect: nếu pick có cả X và Y (revert X) → warn user.

### Release đã tag rồi, commit không vào kịp

Nếu `release/<app>/<v>` đã được tag và deploy prod, **không nên** cherry-pick thêm — phải cut release mới (`v.<X.Y.Z+1>`). Skill check trước khi pick:

```bash
git tag --points-at origin/release/<app>/<version>
```

Nếu có tag pointing tới HEAD của release → warn user: "Release đã tag `<tag>`. Cherry-pick thêm sẽ làm release version khớp với tag không nhất quán. Khuyến nghị cut release mới."

## Out of scope (Maintainer làm tay)

| Flow | Lý do skip |
|---|---|
| Cut release branch mới (`main → release/<app>/<v-mới>`) | Lần đầu cut từ main = FF được, Maintainer làm tay |
| Tag release (`git tag v1.2.0`) + deploy prod | Cần Maintainer judgement + permission |
| Cherry-pick `builds/dev/* → main` | Vi phạm rule 1-chiều — dùng `gitlab-sync audit` thay thế |
| Cherry-pick `release/<app>/* → main` | Vi phạm rule 1-chiều — fix gốc trên main, cherry ngược |
| Merge `main → release/<app>/<v>` (full sync) | Vỡ frozen state của release — chỉ accept cherry-pick có chủ ý |

## Safety rules

- 🚫 **KHÔNG tạo MR `release/<app>/* → main`** dưới bất kỳ hình thức nào. Nếu user yêu cầu, từ chối + giải thích rule 1-chiều
- 🚫 **KHÔNG `git merge main` vào release branch** — sẽ kéo HẾT main về, vỡ frozen state
- 🚫 **KHÔNG `git push --force`** vào `main`, `release/*`, `builds/*` — kể cả `--force-with-lease`
- 🚫 **KHÔNG cherry-pick commit chưa land main** (vd commit từ branch feature chưa merge)
- 🚫 **KHÔNG bỏ flag `-x`** — mất trace SHA gốc, sau audit không biết commit đến từ đâu
- ✅ **Mọi cherry branch là ephemeral** — đảm bảo `--remove-source-branch` khi tạo MR
- ✅ **Cherry-pick theo thứ tự chronological** (cũ → mới) để giảm conflict
- ✅ **Hỏi user trước khi resolve conflict** nếu không chắc bên nào đúng — đặc biệt với file config, migration, route
- ✅ **Verify target branch của MR** trước khi `glab mr create` — đọc lại `--target-branch` 2 lần
- ✅ **Echo lại danh sách commit sẽ pick + chờ xác nhận** trước khi execute (Step 3)
- 🚫 **KHÔNG chèn AI attribution** (Co-Authored-By Claude, 🤖, claude.com) vào: cherry-pick commit message (giữ nguyên message gốc), MR title/description, comment MR

## Tools required

- `git` (luôn có)
- `glab` (GitLab CLI) — cần để tạo MR cherry-pick. Tham khảo install: https://gitlab.com/gitlab-org/cli

## Interaction với gitlab-flow / gitlab-sync

| Tình huống | Dùng skill |
|---|---|
| Feature mới từ Jira task → main | `gitlab-flow` |
| Review/merge MR feature → main | `gitlab-flow` |
| Sau feature merged, deploy QA (sync main → builds/dev) | `gitlab-sync` |
| Audit builds/dev có sạch không | `gitlab-sync` (build hygiene) |
| Backport fix/feature từ main → release để cut patch | `gitlab-cherrypick` (skill này) |
| Cut release branch mới từ main | Out of scope — Maintainer làm tay |
| Tag + deploy prod | Out of scope — Maintainer làm tay |

3 skill độc lập nhưng bổ sung nhau:
- `gitlab-flow` lo "code đi vào main"
- `gitlab-sync` lo "main → builds/dev/<app>" (QA deploy)
- `gitlab-cherrypick` lo "main → release/<app>/<v>" (patch release)
