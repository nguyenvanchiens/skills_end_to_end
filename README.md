# my-skills-gitlab-flow

Bộ skills GitLab workflow cho Claude Code và các AI coding harness khác. Tách ra từ [`nguyenvanchiens/my-skills`](https://github.com/nguyenvanchiens/my-skills) để gọn install khi project chỉ cần workflow GitLab.

## Skills

| Skill | Mô tả | License |
|---|---|---|
| [`gitlab-flow`](skills/gitlab-flow/) | Quy trình end-to-end Jira → branch → commit → MR → review → fix → merge dùng `glab`. Chuẩn hoá branch naming, commit format và safety rules cho team GitLab. Đã tích hợp toàn bộ spec của `commit` và `review-branch` qua các trigger `commit and push` và `review the whole branch`. | MIT |
| [`gitlab-sync`](skills/gitlab-sync/) | Deploy QA cho monorepo multi-app: sync `main → builds/dev/<app>` qua nhánh trung gian `sync/*`, resolve conflict 1 chiều, audit build hygiene. **Chỉ Lead/Maintainer cần cài** — thành viên team không dùng thì không cài để giảm context. | MIT |
| [`gitlab-cherrypick`](skills/gitlab-cherrypick/) | Cherry-pick commit từ `main` vào `release/<app>/<version>` để cut patch release / backport fix. Interactive: list commit main theo số ngày → user pick → cherry-pick qua nhánh `cherry/*` → MR về release branch. **Chỉ Lead/Maintainer cần cài**. | MIT |
| [`commit`](skills/commit/) | Tạo commit theo Conventional Commits, Jira ID ở cuối subject trong ngoặc đơn `(WRA-9)` (`/commit WRA-9`). Tự phân tích diff, chọn `type`/`scope`, có `--quick` mode, partial-staging guard, đọc `.commit-scopes` allowlist. **Standalone** — nếu đã cài `gitlab-flow` thì không cần cài thêm. | MIT |
| [`review-branch`](skills/review-branch/) | Review toàn bộ thay đổi của branch hiện tại so với `main` (committed + uncommitted) qua 3 agent song song: reuse, quality, efficiency — rồi tự fix issue. Standalone — đã tích hợp trong `gitlab-flow`. | MIT |

> **Recommendation**:
> - **Thành viên team**: chỉ cài `gitlab-flow` là đủ cho 3 vai trò (workflow + commit + review).
> - **Lead/Maintainer**: cài thêm `gitlab-sync` (deploy QA) + `gitlab-cherrypick` (patch release) tuỳ nhu cầu.
> - Cài `commit` hoặc `review-branch` riêng chỉ khi không dùng GitLab/`glab`.

## Cài đặt

Yêu cầu: Node.js (để dùng `npx`).

### Cài `gitlab-flow` (recommended — đã bao gồm `commit` + `review-branch`)

```bash
npx skills add nguyenvanchiens/my-skills-gitlab-flow -s gitlab-flow -y -a claude-code --copy
```

### Cài thêm `gitlab-sync` (Lead/Maintainer)

Chỉ Lead/Maintainer cần — dùng để sync `main → builds/dev/<app>` cho deploy QA monorepo multi-app:

```bash
npx skills add nguyenvanchiens/my-skills-gitlab-flow -s gitlab-sync -y -a claude-code --copy
```

### Cài thêm `gitlab-cherrypick` (Lead/Maintainer)

Chỉ Lead/Maintainer cần — dùng để cherry-pick commit `main → release/<app>/<version>` cho cut patch release:

```bash
npx skills add nguyenvanchiens/my-skills-gitlab-flow -s gitlab-cherrypick -y -a claude-code --copy
```

### Cài tất cả 5 skills

```bash
npx skills add nguyenvanchiens/my-skills-gitlab-flow --all -a claude-code --copy
```

### Cài standalone `commit` hoặc `review-branch`

Chỉ dùng nếu KHÔNG dùng GitLab/`glab`:

```bash
npx skills add nguyenvanchiens/my-skills-gitlab-flow -s commit         -y -a claude-code --copy
npx skills add nguyenvanchiens/my-skills-gitlab-flow -s review-branch  -y -a claude-code --copy
```

### Cài global (dùng cho mọi project)

Thêm `-g` vào lệnh, ví dụ:

```bash
npx skills add nguyenvanchiens/my-skills-gitlab-flow -s gitlab-flow -y -g -a claude-code --copy
```

### Cập nhật

```bash
npx skills update         # cập nhật tất cả
npx skills update -g      # chỉ global
```

### Gỡ cài đặt

```bash
npx skills remove
```

## Hỗ trợ harness khác

Cờ `-a` chấp nhận: `claude-code`, `cursor`, `gemini-cli`, `codex`, `opencode`, `windsurf`, `copilot`, ... (xem `npx skills --help` cho danh sách đầy đủ).

```bash
# Cursor
npx skills add nguyenvanchiens/my-skills-gitlab-flow -s gitlab-flow -a cursor --copy

# Tất cả harness phát hiện được
npx skills add nguyenvanchiens/my-skills-gitlab-flow --all -a "*" --copy
```

## Sử dụng `gitlab-flow`

Skill này không phải `/slash command` mà kích hoạt bằng **trigger phrase tiếng Anh** trong prompt thường. Claude tự match phrase và chạy procedure tương ứng.

### Yêu cầu trước khi dùng

- `git` (luôn có)
- [`glab`](https://gitlab.com/gitlab-org/cli) — GitLab CLI. Trên Windows: `winget install GLab.GLab`
- Đăng nhập 1 lần: `glab auth login --hostname <gitlab-host>` (token scope `api` + `write_repository`)

### Bảng trigger

> **Lưu ý copy-paste**: cột Prompt không dùng backtick để tránh GitHub auto-wrap thêm xuống dòng khi copy. Cứ chọn nguyên dòng prompt rồi paste vào Claude Code.

| Prompt | Hành động |
|---|---|
| **create branch from task &lt;TASK-ID&gt;** | Bóc tách task title → đề xuất 1-2 branch ngắn (2-4 từ key, ≤50 chars), **DỪNG hỏi user pick**, pull `main`, rồi mới `git checkout -b feature/<TASK-ID>-<short-desc>` |
| **rename branch &lt;new-name&gt;** | Đổi tên branch hiện tại đồng bộ local + remote. Detect upstream → nếu chưa push: rename thuần. Đã push: rename local + push tên mới + hỏi xóa branch cũ trên remote. Tránh tình trạng local≠remote name làm hỏng push/MR sau đó |
| (paste mô tả task Jira) | Đọc scope, sinh code theo convention project |
| **review the last change** / **review change** | Chạy `git diff`, list issues `#1`, `#2`... |
| **review change simplify** (thêm "simplify" bất kỳ vị trí) | Auto-fix mechanical issues (Reuse/Quality/Efficiency) trước, rồi list review issues |
| **review the whole branch** | Review cumulative branch vs `main` qua 3 agent song song (Reuse / Quality / Efficiency), tự fix issues. **Macro review** trước khi commit cuối / mở MR. |
| **commit and push không push** (kèm `--quick` nếu cần) | Self-contained — kế thừa toàn bộ spec của `/commit`: probe repo, partial-staging guard, atomic check, `.commit-scopes` allowlist, 11 types, footer (`Closes`/`Refs`...), Quick mode, WIP/Spike, revert format. TASK-ID tự lấy từ tên nhánh. Commit local xong **HỎI user** có push không (không tự push). Detect upstream tracking — nếu local branch khác upstream (rename scenario) → STOP, hướng user qua `rename branch`. Không cần cài skill `commit` riêng. |
| **create a merge request** | `glab mr create` với title/description chuẩn |
| **review the MR !&lt;N&gt;** | Lấy `glab mr diff <N>` + comment đã có. **MR chưa có comment** → review mới, list issues + verdict. **MR đã có comment** → review tiếp nối: đối chiếu issue cũ (`✓ Resolved` / `❌ Still open` / `⚠️ Partially`) + chỉ review commit mới push thêm |
| **post review result to the MR** | `glab mr note` đăng comment Markdown |
| **fix all issues** / **fix issue #&lt;N&gt;** | Fix các issue → tóm tắt + đề xuất commit message `fix(<scope>): address review issues #N (<TASK-ID>)` → **HỎI user xác nhận** trước khi commit/push (không tự động) |
| **merge the request** | Check approve + CI pass → `glab mr merge --squash --remove-source-branch` |

### Flow điển hình end-to-end

```
1. create branch from task WRA-40 giới hạn domain account
2. (paste mô tả task)              → Claude code
3. review the last change          → fix nếu cần (lặp 2↔3 nhiều lần)
4. commit and push                 (lặp 2-4 cho từng đoạn)
   ...
5. review the whole branch         → macro review + auto-fix, trước MR
6. commit and push                 → commit fix nếu /review-branch sửa gì
7. create a merge request

   --- chuyển sang vai Reviewer ---

8.  review the MR !21              → Claude in review ra terminal (chưa lên GitLab)
9.  (đọc, chỉnh nếu cần)
10. post review result to the MR   → mới đẩy comment lên GitLab

    --- quay lại vai Developer ---

11. fix all issues                 → fix xong, đợi user xác nhận
12. (xác nhận) commit and push
13. merge the request
```

> **Lưu ý**: bước 8 và 10 là **2 prompt riêng**, không tự động nối. Mục đích để reviewer xem trước nội dung review, có thể yêu cầu Claude bổ sung/sửa, mới quyết định post lên MR.

### Convention

- **Branch**: **default `feature/<TASK-ID>-<short-desc>`** cho mọi loại thay đổi (kể cả bug fix). User override bằng cách tự gõ `bugfix/...` hoặc `hotfix/...` (Mode A — skill respect nguyên si). Desc 2-4 từ key, kebab-case, không dấu, **tổng ≤50 chars**. Drop **type filler** (`Cai-tien`, `Improve`, `Fix`, `Sua`, `Tao`, `Add`, `Create`, `Them`, `Bo-sung`) nhưng **KEEP direction marker** (`Allow`, `Validate`, `Block`, `Duplicate`, `Stale`, `Missing`) **và context marker** (`Show`/`Display`, `Filter`/`Sort`, `Sync`/`Migrate`). Vd `feature/SMT-460-Allow-qty-0-checkin-checkout` (43), bug fix: `feature/HNCW-311-Duplicate-survey-log` (37)
- **Commit**: `<type>(<scope>): <subject> (<TASK-ID>)` (vd `feat(auth): restrict login to allowed domains (WRA-40)`)
- **Target branch**: mặc định `main` — nếu repo dùng `master`, thêm dòng vào `CLAUDE.md` của project: `Default branch: master (not main)`

### Safety rules

- KHÔNG force push vào nhánh đã có MR mở
- KHÔNG merge thẳng vào `main` từ local — luôn qua MR
- KHÔNG bypass hooks (`--no-verify`) trừ khi user yêu cầu rõ
- KHÔNG commit secrets (`.env`, key, token, password)

Xem chi tiết đầy đủ ở [`skills/gitlab-flow/SKILL.md`](skills/gitlab-flow/SKILL.md).

## Sử dụng `gitlab-sync` (deploy QA cho monorepo multi-app)

Skill `gitlab-sync` pair với `gitlab-flow`. Sau khi feature merged main qua `gitlab-flow`, Maintainer dùng `gitlab-sync` để đưa code từ `main` lên các nhánh `builds/dev/<app>` trigger deploy QA — đặc biệt khi 2 nhánh bị conflict.

> **Chỉ Lead/Maintainer cần cài.** Thành viên team không xử lý deploy QA thì không cài để giảm context — tránh load thêm rule + trigger không dùng tới.

### Cài đặt

```bash
npx skills add nguyenvanchiens/my-skills-gitlab-flow -s gitlab-sync -y -a claude-code --copy
```

### Khi nào cần `gitlab-sync`

- Team dùng convention `main → builds/dev/<app>` để trigger CI/CD deploy QA
- Project là **monorepo multi-app** (có nhiều nhánh build dạng `builds/dev/portal-web-admin`, `builds/dev/gift-api`, `builds/dev/portal-api`...)
- `main → builds/dev/<app>` thỉnh thoảng bị conflict, cần resolve mà không leak code `builds/*` ngược về `main`

Nếu team chỉ có 1 build branch hoặc không dùng convention này → không cần `gitlab-sync`.

### Sơ đồ flow (4 bước)

```
main ──────●─────────────●  (giữ nguyên, không đụng vào)
            \
             ↓ (1) tạo sync branch từ main
             ●─────────●  sync/main-to-dev-<app>
                  ↑    ↑
                  │   (3) resolve conflict (giữ phía main) + commit
                  │
                  (2) merge builds/dev/<app> vào sync
                  │
builds/dev/<app> ─●─┘─────●  ← (4) tạo MR sync/* → builds/dev/<app>
```

**Nguyên tắc**: code chảy 1 chiều `main → builds/dev/<app>`. KHÔNG bao giờ PR ngược `builds/* → main`.

### Bảng trigger

| Prompt | Hành động |
|---|---|
| **list build branches** | List tất cả `builds/dev/<app>` có trong repo, để user pick app cần sync |
| **sync main to dev-&lt;app&gt;** | Sync `main → builds/dev/<app>`. Tạo nhánh `sync/main-to-dev-<app>`, merge `builds/dev/<app>` vào, resolve conflict, push, tạo MR. **HỎI user xác nhận** trước khi push |
| **sync main to dev-all** | Sync nhiều app cùng lúc. Loop tuần tự, mỗi app 1 MR riêng, dừng giữa từng app để user confirm |
| **kiểm tra build hygiene** / **audit all dev builds** | Phát hiện vi phạm rule "không commit thẳng `builds/*`". List commit lạ + đề xuất cleanup (cherry-pick về main hoặc reset build) |

### Naming convention

- **Sync branch**: `sync/main-to-dev-<app>` (ephemeral, xoá ngay sau khi MR merged)
- **Commit message**: `chore(sync): resolve conflict main → builds/dev/<app>`
- **MR target**: luôn là `builds/dev/<app>` — KHÔNG bao giờ là `main`

### Out of scope

Skill chỉ tập trung `main → builds/dev/<app>` vì các flow khác (`release → builds/prod`, cut release, cherry-pick hotfix) trong thực tế gần như luôn fast-forward, Maintainer làm tay được. Nếu sau này phát sinh nhu cầu sẽ mở rộng.

### Safety rules

- KHÔNG tạo MR `builds/* → main` dưới bất kỳ hình thức nào
- KHÔNG force push vào `main`/`builds/*` (kể cả `--force-with-lease`) trừ khi user/Maintainer chủ động ra lệnh
- KHÔNG dùng `git checkout --ours <file>` cho code logic mà không đọc qua diff
- Hỏi user trước khi resolve nếu không chắc bên nào đúng (đặc biệt với `.env`, config, route)

Xem chi tiết đầy đủ ở [`skills/gitlab-sync/SKILL.md`](skills/gitlab-sync/SKILL.md).

## Sử dụng `gitlab-cherrypick` (patch release cho monorepo multi-app)

Skill `gitlab-cherrypick` pair với `gitlab-flow`. Khi cần backport fix/feature từ `main` sang `release/<app>/<version>` để cut patch release, skill này lo phần chọn commit + cherry-pick + tạo MR an toàn qua nhánh trung gian.

> **Chỉ Lead/Maintainer cần cài.** Thành viên team không xử lý patch release thì không cài để giảm context — tránh load thêm rule + trigger không dùng tới.

### Cài đặt

```bash
npx skills add nguyenvanchiens/my-skills-gitlab-flow -s gitlab-cherrypick -y -a claude-code --copy
```

### Khi nào cần `gitlab-cherrypick`

- Team có release branch dạng `release/<app>/<version>` (vd `release/portal-web-admin/v1.2.0`, `release/gift-api/v0.5.3`)
- Cần đưa 1 vài commit từ `main` vào release đã cut để patch (`v1.2.1`, `v0.5.4`...)
- Muốn workflow interactive: list commit main theo N ngày → pick → execute, thay vì phải nhớ SHA

Nếu team không dùng convention release branch riêng cho từng version → không cần `gitlab-cherrypick`.

### Sơ đồ flow (4 bước)

```
main ──●──●──●──●──●  (giữ nguyên, không đụng vào)
       │  │     │
       │  │     │   (1) tạo cherry branch từ release target
       │  │     ↓
       │  │     ●──●  cherry/main-to-release-<app>-<v>
       │  │           ↑
       │  │           (2) cherry-pick -x các commit user đã pick (chronological order)
       │  │           ↑
       │  │           (3) resolve conflict nếu có
       │  │
release/<app>/<v> ●───────●  ← (4) tạo MR cherry/* → release/<app>/<v>
```

**Nguyên tắc**: code chảy 1 chiều `main → release/<app>/<v>`. KHÔNG bao giờ PR ngược `release/* → main`.

### Bảng trigger

| Prompt | Hành động |
|---|---|
| **list releases** | List tất cả `release/<app>/<v>` có trong repo, group theo app, để user pick release cần patch |
| **list commit main last &lt;N&gt; days** | List commit trên `main` trong N ngày qua (không cần release target). Khảo sát trước khi pick |
| **cherry-pick to release/&lt;app&gt;/&lt;v&gt;** | Flow chính: hỏi N ngày → list commit main (đã loại trừ commit có sẵn trên release) → user pick theo `#` → confirm → tạo `cherry/main-to-release-<app>-<v>` từ release branch → `cherry-pick -x` chronological order → resolve conflict → **HỎI user xác nhận** trước khi push → tạo MR target = release branch |

### Naming convention

- **Cherry branch**: `cherry/main-to-release-<app>-<version>` (ephemeral, xoá ngay sau khi MR merged). Slash trong release name (`release/<app>/<v>`) được dash hoá khi đưa vào cherry branch
- **Commit sau cherry-pick**: giữ NGUYÊN message gốc + dòng `(cherry picked from commit <SHA>)` (auto thêm bởi `git cherry-pick -x`)
- **MR title**: `chore(cherry): backport <N> commit từ main → release/<app>/<version>`
- **MR target**: luôn là `release/<app>/<version>` — KHÔNG bao giờ là `main`, `builds/dev/*`

### Out of scope

Skill chỉ tập trung `main → release/<app>/<v>`. Cut release branch mới (`main → release/<app>/<v-mới>`), tag release, deploy prod — Maintainer làm tay vì cần judgement + permission cao.

### Safety rules

- KHÔNG tạo MR `release/<app>/* → main` dưới bất kỳ hình thức nào
- KHÔNG `git merge main` vào release branch — sẽ vỡ frozen state, phải cherry-pick có chủ ý
- KHÔNG bỏ flag `-x` của `git cherry-pick` — mất trace SHA gốc
- KHÔNG cherry-pick commit chưa land main (vd từ feature branch chưa merge)
- Cherry-pick theo thứ tự chronological (cũ → mới) để giảm conflict
- Echo lại danh sách commit sẽ pick + chờ xác nhận trước khi execute

Xem chi tiết đầy đủ ở [`skills/gitlab-cherrypick/SKILL.md`](skills/gitlab-cherrypick/SKILL.md).

## Cấu trúc repo

```
my-skills-gitlab-flow/
├── README.md
└── skills/
    ├── gitlab-flow/
    │   └── SKILL.md
    ├── gitlab-sync/
    │   └── SKILL.md
    ├── gitlab-cherrypick/
    │   └── SKILL.md
    ├── commit/
    │   └── SKILL.md
    └── review-branch/
        └── SKILL.md
```

Mỗi `SKILL.md` có frontmatter `name` + `description` để CLI `npx skills` tự nhận diện.

## License

MIT.
