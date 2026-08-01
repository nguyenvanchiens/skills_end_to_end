# skills_end_to_end

Bộ skills GitLab workflow cho Claude Code và các AI coding harness khác. Tách ra từ [`nguyenvanchiens/my-skills`](https://github.com/nguyenvanchiens/my-skills) để gọn install khi project chỉ cần workflow GitLab.

## Skills

| Skill | Mô tả | License |
|---|---|---|
| [`gitlab-flow`](skills/gitlab-flow/) | Quy trình vai Developer: Jira → branch → code → review nhẹ → commit → MR → fix, dùng `glab`. Chuẩn hoá branch naming, commit format và safety rules cho team GitLab. Đã tích hợp toàn bộ spec của `commit` qua trigger `commit and push`. Phần vai Reviewer tách sang `gitlab-review`. | MIT |
| [`gitlab-review`](skills/gitlab-review/) | **Add-on cho Lead** — vai Reviewer: `review the whole branch` (4 agent chuyên biệt + dedup/verify), `review the MR !N` (mức Grounded, đọc full file tại revision MR mà không cần checkout), `post review result to the MR`, `merge the request`. **Yêu cầu cài kèm `gitlab-flow`** — tham chiếu conventions của nó, không chép lại. | MIT |
| [`gitlab-sync`](skills/gitlab-sync/) | Deploy QA cho monorepo multi-app: sync `main → builds/dev/<app>` qua nhánh trung gian `sync/*`, resolve conflict 1 chiều, audit build hygiene. **Chỉ Lead/Maintainer cần cài** — thành viên team không dùng thì không cài để giảm context. | MIT |
| [`gitlab-cherrypick`](skills/gitlab-cherrypick/) | Cherry-pick commit từ `main` vào `release/<app>/<version>` để cut patch release / backport fix. Interactive: list commit main theo số ngày → user pick → cherry-pick qua nhánh `cherry/*` → MR về release branch. **Chỉ Lead/Maintainer cần cài**. | MIT |
| [`commit`](skills/commit/) | ⚠️ **Deprecated** — spec đã gộp vào `gitlab-flow` (trigger `commit and push`) để tránh trùng lặp + 2 mô hình invoke. File chỉ còn là con trỏ. Dùng `commit and push` của `gitlab-flow` thay thế (kể cả khi không dùng `glab` — commit là local, push được gate). | MIT |
| [`review-branch`](skills/review-branch/) | ⚠️ **Deprecated** — gộp vào `gitlab-review`. File chỉ còn là con trỏ. | MIT |

> **Recommendation**:
> - **Thành viên team**: chỉ cài `gitlab-flow`. Bộ này ~5.250 từ; cài thêm phần Reviewer sẽ đội lên ~8.150 từ mỗi lần trigger mà bạn không dùng tới.
> - **Lead/Maintainer**: `gitlab-flow` + `gitlab-review`, thêm `gitlab-sync` / `gitlab-cherrypick` tuỳ nhu cầu.
> - `commit` và `review-branch` đã deprecated (gộp vào `gitlab-flow` / `gitlab-review`).

## Cài đặt

Yêu cầu: Node.js (để dùng `npx`).

### 📦 Bộ Dev — mọi thành viên

```bash
npx skills add nguyenvanchiens/skills_end_to_end -s gitlab-flow -y -a claude-code --copy
```

Đủ cho: tạo branch từ task → sinh code → review nhẹ → commit → mở MR → fix review.

### 📦 Bộ Lead / Maintainer

```bash
npx skills add nguyenvanchiens/skills_end_to_end -s gitlab-flow -y -a claude-code --copy
npx skills add nguyenvanchiens/skills_end_to_end -s gitlab-review -y -a claude-code --copy
npx skills add nguyenvanchiens/skills_end_to_end -s gitlab-sync -y -a claude-code --copy
npx skills add nguyenvanchiens/skills_end_to_end -s gitlab-cherrypick -y -a claude-code --copy
```

⚠️ Dòng `gitlab-flow` **bắt buộc** — `gitlab-review` là add-on tham chiếu conventions của nó, cài thiếu sẽ hỏng ngầm.

`gitlab-sync` (deploy QA) và `gitlab-cherrypick` (patch release) tuỳ nhu cầu, bỏ được.

### Cập nhật

```bash
npx skills update
```

Chạy trong project. Cả hai skill về cùng một commit nên `gitlab-review` luôn khớp conventions của `gitlab-flow`.

## Hỗ trợ harness khác

Cờ `-a` chấp nhận: `claude-code`, `cursor`, `gemini-cli`, `codex`, `opencode`, `windsurf`, `copilot`, ... (xem `npx skills --help` cho danh sách đầy đủ).

```bash
# Cursor
npx skills add nguyenvanchiens/skills_end_to_end -s gitlab-flow -a cursor --copy

# Tất cả harness phát hiện được
npx skills add nguyenvanchiens/skills_end_to_end --all -a "*" --copy
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
| **create branch from task &lt;TASK-ID&gt;** | Bóc tách task title → đề xuất 1-2 branch ngắn (2-4 từ key, ≤50 chars), **DỪNG hỏi user pick tên branch**. Tiếp đó **HỎI user chọn base branch** (`main` hay `dev`/`develop`...) — trừ khi user đã nói rõ trong prompt (vd "base từ dev"). Dù chọn base nào cũng **luôn `git fetch` + `git pull`** lấy code mới nhất rồi mới `git checkout -b feature/<TASK-ID>-<short-desc>` |
| **rename branch &lt;new-name&gt;** | Đổi tên branch hiện tại đồng bộ local + remote. Detect upstream → nếu chưa push: rename thuần. Đã push: rename local + push tên mới + hỏi xóa branch cũ trên remote. Tránh tình trạng local≠remote name làm hỏng push/MR sau đó |
| (paste mô tả task Jira) | Đọc scope, sinh code theo convention project |
| **review the last change** / **review change** | Soi diff gần nhất: **đọc full file đã đổi + CLAUDE.md/file lân cận + task** để ground (không review diff "ống hút"), **verify lọc false positive**, rồi list issues `#1`, `#2`... kèm `file:line`. Inline (nhẹ) — diff lớn/cần sâu → `review the whole branch` hoặc `/code-review` |
| **review change simplify** (thêm "simplify" bất kỳ vị trí) | Auto-fix mechanical issues (Quality & Reuse + Efficiency) trước, rồi list review issues |
| **review the whole branch** | Review cumulative branch vs `<base>` (main/dev đã chốt lúc tạo branch) qua **4 agent song song** (Correctness/Task-fit · Security · Efficiency · Quality & Reuse), mỗi agent bắt buộc đọc full file + truy caller, rồi **dedup + verify** trước khi auto-fix `Blocker`/`Major`. **Macro review** trước khi commit cuối / mở MR. |
| **commit and push** (kèm `--quick` nếu cần) | Self-contained — kế thừa toàn bộ spec commit: probe repo, partial-staging guard, atomic check, `.commit-scopes` allowlist, 11 types, footer (`Closes`/`Refs`...), Quick mode, WIP/Spike, revert format. TASK-ID tự lấy từ tên nhánh. Commit local xong **HỎI user** có push không (**không tự push** dù tên trigger có "push"). Detect upstream tracking — nếu local branch khác upstream (rename scenario) → STOP, hướng user qua `rename branch`. (Skill `commit` riêng đã deprecated — dùng trigger này.) |
| **create a merge request** | Xác định **target branch** trước: nếu đã chốt base trong session thì dùng, **chưa biết (vd MR ở session khác) → HỎI `main` hay `dev`**, không mặc định `main`. Rồi `glab mr create --target-branch <base>` với title/description chuẩn |
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

### Team review policy (tiết kiệm token cho gói Pro)

Áp dụng cho team **mixed plan**: đa số member dùng **Claude Pro** (usage cap thấp), reviewer chính dùng **Pro Max** (cap cao). Mục tiêu: member không cháy token, lượt review nặng dồn vào tài khoản Max.

| Vai | Plan | Làm gì khi review | Tránh |
|---|---|---|---|
| **Member (tác giả)** | Pro | Chỉ `review the last change` cho mỗi đoạn sửa (nhẹ, inline, 1 context) → commit → mở MR | **KHÔNG** cài `gitlab-review` (4 agent — đốt token nhiều, và Lead đã lo phần này) |
| **Reviewer chính** | Pro Max | Sau khi member mở MR, chạy `review the MR !<ID>` — mặc định chạy ở mức **Grounded**: `glab mr diff` + `git fetch` rồi đọc full file qua `git show FETCH_HEAD:<file>`, **không cần checkout**, không đụng working tree. MR quan trọng cần soi sâu → `glab mr checkout <ID>` rồi `review the whole branch` (4-agent) | — |

**Nguyên tắc**: bước review nặng (4-agent) **chỉ chạy ở phía reviewer Pro Max** — token tính trên tài khoản đó, member Pro không ảnh hưởng. Member chỉ tốn token cho review inline hằng ngày.

> ⚠️ `review the MR !<ID>` chạy được mà **không cần đứng trên nhánh MR**. Mặc định nó chạy ở mức **Grounded**: `glab mr diff` lấy diff, rồi `git fetch origin <source-branch>` + `git show FETCH_HEAD:<file>` để đọc **full file tại đúng revision của MR** — không đụng working tree, nên bạn vẫn code dở ở nhánh khác được. Muốn soi SÂU hơn (4-agent ensemble + verify) thì **phải đứng trên nhánh source của MR trước** — checkout bằng tool nào cũng được (`glab mr checkout <ID>`, `git checkout <branch>`, hay **SourceTree/GUI**); skill chỉ cần HEAD đang ở đúng nhánh. `review the whole branch` và `/code-review` đọc code local, không soi được remote MR khi bạn đang ở nhánh khác.
>
> **Base khi deep-review = target branch của MR** (`glab mr view <ID>`), không cần nhớ/đoán `main` hay `dev`: dự án main-chính → MR target `main`, dự án dev-chính → target `dev`. Skill tự đọc target từ MR.

> Lý do bỏ được `review the whole branch` ở phía member: lượt `review the MR !<ID>` của reviewer đã soát toàn bộ diff MR (full coverage). Member review sớm bằng `review the last change` chỉ để bắt lỗi rẻ ngay khi code, không cần lượt 4-agent.

### Review output

Cả 3 trigger review (`review the last change`, `review the whole branch`, `review the MR !<N>`) đều gán **severity** cho mỗi finding:

| Severity | Hành động |
|---|---|
| `Blocker` (sai logic, security, mất data, crash) · `Major` (edge case thật, N+1 hot path, race) | **Fix** |
| `Minor` (naming, code thừa, abstraction) | Chỉ liệt kê — gõ `fix issue #N` nếu muốn fix |
| `Nit` (style, ý kiến cá nhân) | Bỏ, không báo |

Không có Blocker/Major → skill trả **"Không có vấn đề chặn"** và dừng, không bịa thêm cho đủ danh sách. Đây là chủ đích: LLM reviewer có bias luôn phải tìm ra cái gì đó, khiến review sau fix lại ra issue mới không hồi kết. Mỗi finding bắt buộc có `file:line` + code đã đọc thật.

### Convention

- **Branch**: **default `feature/<TASK-ID>-<short-desc>`** cho mọi loại thay đổi (kể cả bug fix). User override bằng cách tự gõ `bugfix/...` hoặc `hotfix/...` (Mode A — skill respect nguyên si). Desc 2-4 từ key, kebab-case, không dấu, **tổng ≤50 chars**. Drop **type filler** (`Cai-tien`, `Improve`, `Fix`, `Sua`, `Tao`, `Add`, `Create`, `Them`, `Bo-sung`) nhưng **KEEP direction marker** (`Allow`, `Validate`, `Block`, `Duplicate`, `Stale`, `Missing`) **và context marker** (`Show`/`Display`, `Filter`/`Sort`, `Sync`/`Migrate`). Vd `feature/SMT-460-Allow-qty-0-checkin-checkout` (43), bug fix: `feature/HNCW-311-Duplicate-survey-log` (37)
- **Commit**: `<type>(<scope>): <subject> (<TASK-ID>)` (vd `feat(auth): restrict login to allowed domains (WRA-40)`)
- **Base branch** (gốc tạo nhánh + target MR + merge-base review): mặc định `main`. Project tạo nhánh từ `dev`/`develop`/`master` → khi gõ `create branch from task`, skill **hỏi chọn `main` hay `dev`** rồi dùng nhất quán cho cả vòng đời branch (tạo nhánh, target MR, review). Dù chọn base nào cũng luôn `fetch` + `pull` mới nhất trước khi tạo. Có thể nói rõ trong prompt ("base từ dev") để khỏi hỏi.

### Safety rules

- KHÔNG force push vào nhánh đã có MR mở
- KHÔNG merge thẳng vào `<base>` (main/dev) từ local — luôn qua MR
- KHÔNG bypass hooks (`--no-verify`) trừ khi user yêu cầu rõ
- KHÔNG commit secrets (`.env`, key, token, password)

Xem chi tiết đầy đủ ở [`skills/gitlab-flow/SKILL.md`](skills/gitlab-flow/SKILL.md).

## Sử dụng `gitlab-sync` (deploy QA cho monorepo multi-app)

Skill `gitlab-sync` pair với `gitlab-flow`. Sau khi feature merged main qua `gitlab-flow`, Maintainer dùng `gitlab-sync` để đưa code từ `main` lên các nhánh `builds/dev/<app>` trigger deploy QA — đặc biệt khi 2 nhánh bị conflict.

> **Chỉ Lead/Maintainer cần cài.** Thành viên team không xử lý deploy QA thì không cài để giảm context — tránh load thêm rule + trigger không dùng tới.

### Cài đặt

```bash
npx skills add nguyenvanchiens/skills_end_to_end -s gitlab-sync -y -a claude-code --copy
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
npx skills add nguyenvanchiens/skills_end_to_end -s gitlab-cherrypick -y -a claude-code --copy
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
| **cherry-pick to release/&lt;app&gt;/&lt;v&gt;** | Flow chính, **interactive**: (0) **chọn nhánh release target** — gõ đủ `release/<app>/<v>` thì dùng luôn, thiếu thì skill **in danh sách release đánh số → pick `#`** → (1) **chọn source branch** (`main`/`dev` — nhánh tích hợp để lấy commit) + hỏi N ngày → (2) list commit `<source>` (đã loại trừ commit có sẵn trên release) → (3) **chọn commit theo `#`** (skill in kèm cú pháp `2,4` / `1-3` / `all`) → confirm → tạo `cherry/<source>-to-release-<app>-<v>` từ release branch → `cherry-pick -x` chronological → resolve conflict → **HỎI user xác nhận** trước khi push → tạo MR target = release branch |

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
skills_end_to_end/
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

## Skill installation hygiene

> Rule cho Claude khi diagnose/fix vấn đề skill (stale, missing behavior, sync issue) — không liên quan workflow GitLab.

🚫 **KHÔNG tự copy/sync skill file vào `C:\Users\<user>\.claude\skills\` (global) trừ khi user yêu cầu rõ ràng.** Cùng nguyên tắc cho mọi system-level location: `~/.claude/`, `%APPDATA%/Claude/`.

**Default khi user báo skill bị stale/sai**:
1. Verify trong repo `skills_end_to_end` đã có version đúng
2. Gợi ý user chạy `npx skills update` trong project bị ảnh hưởng (KHÔNG `-g`)
3. Gợi ý restart Claude session để load skill mới
4. **Chỉ copy thủ công tới global NẾU user yêu cầu rõ** (vd "sync luôn global đi")

**Lý do**: dual-location dễ tạo state lệch nhau; project-only là single source of truth; user có quyền chọn nơi cài, auto-touch global bypass quyền đó.
