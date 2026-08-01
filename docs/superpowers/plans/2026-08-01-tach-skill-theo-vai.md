# Tách skill theo vai Dev/Lead — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Tách `gitlab-flow` thành bộ Dev (`gitlab-flow`) và bộ Lead (`gitlab-review`) để member chỉ nạp ~5.250 từ thay vì 9.221 từ mỗi task.

**Architecture:** Một repo, hai skill, phụ thuộc một chiều — `gitlab-review` là add-on tham chiếu conventions của `gitlab-flow`, không chép lại. Nội dung tra cứu của `Commit and push` tách ra `commit-reference.md` nạp theo nhu cầu.

**Tech Stack:** Markdown + YAML frontmatter. Không có build/test framework — verification bằng `git`, `grep`, và script đếm từ PowerShell.

**Spec:** [`docs/superpowers/specs/2026-08-01-tach-skill-theo-vai-design.md`](../specs/2026-08-01-tach-skill-theo-vai-design.md)

## Global Constraints

- Repo: `D:\TemplateAi\Project User Main\skills_end_to_end`, branch `main`.
- Ngôn ngữ nội dung: **tiếng Việt** cho `gitlab-flow` / `gitlab-review` / README; **tiếng Anh** cho `review-branch` (giữ nguyên ngôn ngữ gốc từng file).
- Commit format: `<type>(<scope>): <subject>` — **không có TASK-ID** (repo này không dùng Jira; 20 commit gần nhất đều không có).
- Scope hợp lệ (từ `git log`): `gitlab-flow`, `readme`, `review`, `gitlab-cherrypick`, cộng scope mới `gitlab-review`.
- 🚫 **Tuyệt đối không chèn `Co-Authored-By: Claude`, `🤖 Generated with`, hay bất kỳ AI attribution nào** vào commit message. Repo không track AI authorship.
- Không đổi hành vi của bất kỳ trigger nào, **trừ những thay đổi bắt buộc do việc tách**: member cài bộ Dev không còn 4 trigger Lead, nên mọi hướng dẫn trỏ tới chúng phải nói đúng sự thật (kèm tên skill `gitlab-review`, hoặc "nhờ Lead chạy"). Ngoài phạm vi đó, đây là restructure thuần — cùng một nội dung, chỗ khác.
- **Trùng lặp verbatim trong `gitlab-review` là CỐ Ý, đã được phán.** Hai block (severity + grounding) và dòng lens chép vào prompt subagent trùng với `Conventions` ở `gitlab-flow` — bắt buộc, vì subagent chạy context riêng và không thấy `Conventions`. Reviewer báo "trùng lặp logic block" ⇒ finding hợp lệ nhưng **plan thắng**; xử lý bằng dấu đồng bộ ở cả hai đầu, không bằng cách bỏ trùng.
- Không nén/viết lại nội dung ở đợt này.
- Mỗi task kết thúc bằng 1 commit.

## File Structure

| File | Trách nhiệm | Task |
|---|---|---|
| `skills/gitlab-flow/SKILL.md` | Conventions dùng chung + 7 trigger Dev | 2, 3, 4, 6 |
| `skills/gitlab-flow/commit-reference.md` | **Tạo mới** — bảng tra cứu commit (type, footer, scope, quick, WIP, ví dụ) | 4 |
| `skills/gitlab-review/SKILL.md` | **Tạo mới** — 4 trigger Lead, tham chiếu conventions của `gitlab-flow` | 3 |
| `skills/review-branch/SKILL.md` | Rút thành con trỏ tới `gitlab-review` | 5 |
| `README.md` | Đóng gói 2 bộ, bảng skill, hygiene | 6 |
| `skills/TESTS.md` | T9 / T10 / T11 | 7 |

Không đụng: `skills/gitlab-sync/`, `skills/gitlab-cherrypick/`, `skills/commit/`.

## Công cụ verification dùng lại nhiều lần

Lưu đoạn này, các task sẽ gọi lại. Chạy bằng tool PowerShell.

```powershell
# do-tu.ps1 — đếm từ theo section của 1 SKILL.md
param([string]$f)
$lines = Get-Content $f
$total = $lines.Count
$marks = @()
for ($i = 0; $i -lt $total; $i++) {
  if ($lines[$i] -match '^#{2,3} ') { $marks += [pscustomobject]@{ Line = $i + 1; Title = $lines[$i] } }
}
for ($j = 0; $j -lt $marks.Count; $j++) {
  $start = $marks[$j].Line
  if ($j -lt $marks.Count - 1) { $end = $marks[$j + 1].Line - 1 } else { $end = $total }
  $words = ($lines[($start - 1)..($end - 1)] -join ' ' -split '\s+').Count
  [pscustomobject]@{ Tu = $words; Muc = $marks[$j].Title }
}
"TONG TU: " + (($lines -join ' ' -split '\s+').Count)
```

---

### Task 1: Commit công việc 4-agent đang dở

Working tree đang có 4 file sửa từ đợt nâng cấp review pipeline. Phải commit trước, nếu không diff của việc tách sẽ lẫn với việc nâng cấp và không ai review nổi.

**Files:**
- Modify: `README.md`, `skills/TESTS.md`, `skills/gitlab-flow/SKILL.md`, `skills/review-branch/SKILL.md` (đã sửa sẵn, chỉ commit)

**Interfaces:**
- Consumes: không
- Produces: baseline sạch cho mọi task sau. Bảng 4 lens nằm ở `skills/gitlab-flow/SKILL.md` mục `**Phase 2 — Launch 4 agent SONG SONG**`, là nguồn để Task 2 trích ra.

- [ ] **Step 1: Xác nhận đúng 4 file, không dư**

```bash
cd "/d/TemplateAi/Project User Main/skills_end_to_end"
git status --short
```

Kỳ vọng đúng 4 dòng: `M README.md`, `M skills/TESTS.md`, `M skills/gitlab-flow/SKILL.md`, `M skills/review-branch/SKILL.md`. Có dòng thứ 5 ⇒ DỪNG, hỏi user.

- [ ] **Step 2: Kiểm tra không có AI attribution lọt vào nội dung**

```bash
git diff | grep -iE "co-authored-by|generated with|claude\.com|noreply@anthropic"
```

Kỳ vọng: **không output gì**. Có match ⇒ xoá dòng đó khỏi file trước khi commit.

- [ ] **Step 3: Commit**

```bash
cd "/d/TemplateAi/Project User Main/skills_end_to_end"
git add README.md skills/TESTS.md skills/gitlab-flow/SKILL.md skills/review-branch/SKILL.md
git commit -m "$(cat <<'EOF'
feat(review): 4-lens ensemble + tầng verify cho review the whole branch

Thêm 2 agent Correctness/Task-fit và Security. Roster cũ
Reuse/Quality/Efficiency không ai phụ trách Blocker (sai logic vs
task, lỗ hổng security) — tức category duy nhất chặn merge bị bỏ
trống, còn 2/3 agent chỉ sinh Minor vốn không auto-fix.

Bắt agent nạp context trước khi flag (full file, convention repo,
truy caller ngoài diff) và thêm Phase 2.5 dedup + verify mặc định
bác bỏ, tránh auto-fix theo finding suy diễn từ diff.

review the MR mặc định lên mức Grounded: fetch + git show
FETCH_HEAD:<file> đọc full file tại revision của MR, không cần
checkout.
EOF
)"
```

- [ ] **Step 4: Xác nhận working tree sạch**

```bash
git status --short
```

Kỳ vọng: không output gì.

---

### Task 2: Trích danh mục `Review lenses` lên Conventions

Bảng 4 lens hiện nằm trong `**Phase 2**` của `review the whole branch`. Task 3 sẽ chuyển section đó sang `gitlab-review`, nhưng `review change simplify` (ở lại bộ Dev) đang tham chiếu tới nó — nên phải nâng bảng lên `Conventions` trước, thành interface độc lập.

**Files:**
- Modify: `skills/gitlab-flow/SKILL.md`

**Interfaces:**
- Consumes: bảng 4 lens trong `**Phase 2 — Launch 4 agent SONG SONG**` (Task 1)
- Produces: section `### Review lenses` trong `## Conventions`, đặt **ngay sau** `### Review output (áp dụng cho MỌI trigger review)`. Task 3 (`gitlab-review`) và `review change simplify` đều trỏ vào tên section này. Không đổi tên section sau khi tạo.

- [ ] **Step 1: Xác định vị trí chèn**

```powershell
Select-String -Path "D:\TemplateAi\Project User Main\skills_end_to_end\skills\gitlab-flow\SKILL.md" -Pattern '^### Review output|^## Triggers & Procedures|^\| \*\*Correctness'
```

Ghi lại số dòng của `### Review output ...` và `## Triggers & Procedures`. Section mới chèn vào giữa hai mốc đó.

- [ ] **Step 2: Chèn section `### Review lenses`**

Chèn ngay trước dòng `## Triggers & Procedures`:

```markdown
### Review lenses

Danh mục lens dùng chung. Hai nơi tiêu thụ bảng này:

- `review change simplify` (skill này) — dùng **2 lens cuối** (`Efficiency`, `Quality & Reuse`). Đây là pass dọn dẹp, không phải pass tìm bug.
- `review the whole branch` (skill **`gitlab-review`**, chỉ Lead cài) — dùng **cả 4**, mỗi lens một agent.

⚠️ Đây là **nguồn duy nhất** của định nghĩa lens. `gitlab-review` chép từng dòng vào prompt subagent chứ không giữ bản sao — sửa ở đây là sửa cho cả hai.

| Lens | Tập trung | Severity chủ đạo | Flag điển hình |
|---|---|---|---|
| **Correctness / Task-fit** | Code có làm đúng task không | Blocker, Major | Lệch yêu cầu task, thiếu case so với spec, làm dư ngoài scope, edge case (null/empty/list rỗng/boundary/số âm/unicode), off-by-one, `catch` nuốt lỗi, state nửa vời khi throw giữa chừng, migration không idempotent, timezone/rounding |
| **Security** | Lỗ hổng | Blocker | Thiếu input validation, authz/authn bypass (chặn ở UI mà không chặn ở API), SQL/command injection, path traversal, mass-assignment, secret/token/PII lộ ra log hay response, SSRF, CORS/cookie/session sai |
| **Efficiency** | Performance / resource | Major | N+1, missed concurrency (independent ops chạy tuần tự), hot-path bloat, no-op updates trong polling loops, unnecessary existence checks (TOCTOU), unbounded memory, listener leak, overly broad reads |
| **Quality & Reuse** | Code sạch / tái dùng | Minor | New function duplicates existing helper, inline logic could use existing util (string manipulation, path handling, env checks, type guards), redundant state, parameter sprawl, copy-paste với biến thể nhỏ, leaky abstraction, stringly-typed (raw strings nơi đã có enum/constant), unnecessary JSX nesting, nested conditionals 3+ levels, comment giải thích WHAT |
```

- [ ] **Step 3: Trong Phase 2, thay bảng lens bằng con trỏ**

Xoá bảng 4 dòng lens trong `**Phase 2 — Launch 4 agent SONG SONG**` và thay bằng:

```markdown
| Agent | Lens |
|---|---|
| Agent 1 | `Correctness / Task-fit` |
| Agent 2 | `Security` |
| Agent 3 | `Efficiency` |
| Agent 4 | `Quality & Reuse` |

Định nghĩa đầy đủ từng lens (Tập trung · Severity chủ đạo · Flag điển hình): xem **`Review lenses`** ở mục `Conventions`. **Chép nguyên văn dòng của lens tương ứng** vào prompt agent đó — subagent không thấy `Conventions`.
```

Giữ nguyên block ghi chú `> **Vì sao roster là 4 agent này**: ...` bên dưới bảng.

- [ ] **Step 4: Sửa tham chiếu ở `review change simplify` Step 0**

Tìm dòng bắt đầu bằng `2. Scan diff theo 2 góc nhìn` và thay toàn bộ dòng đó bằng:

```markdown
2. Scan diff theo 2 lens **Efficiency** + **Quality & Reuse** — định nghĩa và danh sách flag xem mục **`Review lenses`** ở `Conventions`. Bỏ qua 2 lens `Correctness` và `Security`: đây là pass **dọn dẹp**, không phải pass tìm bug. Inline Claude, không spawn agent vì scope hẹp.
```

- [ ] **Step 5: Verify không còn tham chiếu gãy**

```powershell
Select-String -Path "D:\TemplateAi\Project User Main\skills_end_to_end\skills\gitlab-flow\SKILL.md" -Pattern 'bảng Phase 2|dòng cùng tên'
```

Kỳ vọng: **không match**. Còn match ⇒ còn chỗ trỏ vào bảng đã dời.

```powershell
Select-String -Path "D:\TemplateAi\Project User Main\skills_end_to_end\skills\gitlab-flow\SKILL.md" -Pattern '^### Review lenses'
```

Kỳ vọng: đúng **1** match.

- [ ] **Step 6: Commit**

```bash
cd "/d/TemplateAi/Project User Main/skills_end_to_end"
git add skills/gitlab-flow/SKILL.md
git commit -m "$(cat <<'EOF'
refactor(gitlab-flow): nâng danh mục Review lenses lên Conventions

review change simplify đang trỏ vào bảng lens nằm trong Phase 2 của
review the whole branch. Sắp tách Phase 2 sang skill gitlab-review nên
tham chiếu đó sẽ gãy với member chỉ cài bộ Dev.

Tách bảng thành interface độc lập ở Conventions, hai bên cùng tiêu thụ.
Phase 2 chỉ còn ánh xạ agent -> tên lens.
EOF
)"
```

---

### Task 3: Tạo skill `gitlab-review`, chuyển 4 trigger Lead sang

**Files:**
- Create: `skills/gitlab-review/SKILL.md`
- Modify: `skills/gitlab-flow/SKILL.md` (xoá 4 section, sửa cross-reference)

**Interfaces:**
- Consumes: `### Review lenses`, `### Review output`, `### Base branch`, `### Output language`, `## Safety rules` ở `gitlab-flow` (Task 2)
- Produces: skill `gitlab-review` với 4 section `### "review the whole branch" ...`, `### "review the MR !<N>" ...`, `### "post review result to the MR"`, `### "merge the request"`. Task 5 (`review-branch` → con trỏ) và Task 6 (README) trỏ vào tên skill này.

- [ ] **Step 1: Trích 4 section ra file tạm để không mất chữ nào**

```powershell
$src = "D:\TemplateAi\Project User Main\skills_end_to_end\skills\gitlab-flow\SKILL.md"
Select-String -Path $src -Pattern '^### "review the whole branch"|^### "review the MR|^### "post review result|^### "merge the request"|^### "create a merge request"|^## Safety rules'
```

Ghi lại số dòng. Bốn khối cần chuyển: từ `### "review the whole branch"` tới ngay trước `### "create a merge request"`, và từ `### "review the MR !<N>"` tới ngay trước `## Safety rules`.

⚠️ `### "create a merge request"` và `### "fix all issues"` **ở lại** `gitlab-flow` — chúng nằm xen giữa các khối Lead, phải cắt chính xác.

- [ ] **Step 2: Tạo `skills/gitlab-review/SKILL.md` với frontmatter + header**

```markdown
---
name: gitlab-review
description: Use when reviewing a GitLab merge request or a whole feature branch as the reviewer, posting review results back to the MR, or merging an approved MR. Triggers include "review the whole branch", "review the MR !N", "post review result to the MR", "merge the request".
---

# GitLab Review (vai Reviewer / Lead)

> ## ⚠️ REQUIRES `gitlab-flow`
>
> Skill này là **add-on**. Nó tham chiếu `Review lenses`, `Review output` (bảng severity), `Base branch`, `Output language` và `Safety rules` từ skill **`gitlab-flow`** — và **không** chép lại chúng.
>
> Không thấy các mục đó trong context ⇒ **DỪNG**, báo user cài `gitlab-flow` trước:
> ```bash
> npx skills add nguyenvanchiens/skills_end_to_end -s gitlab-flow -y -a claude-code --copy
> ```
> 🚫 **Tuyệt đối không tự suy ra nội dung thiếu.** Bịa lại bảng severity hay danh mục lens sẽ cho ra review sai chuẩn mà không ai phát hiện.

**Output language**: mặc định tiếng Việt cho cả 4 trigger ở đây, kể cả khi user gõ trigger bằng tiếng Anh — theo mục `Output language` ở `gitlab-flow`.

**Severity**: mọi finding phải gán `Blocker` / `Major` / `Minor` / `Nit` theo bảng `Review output` ở `gitlab-flow`. Danh sách rỗng là kết quả hợp lệ.

## Triggers & Procedures
```

- [ ] **Step 3: Dán 4 section đã trích vào sau `## Triggers & Procedures`**

Thứ tự: `review the whole branch` → `review the MR !<N>` → `post review result to the MR` → `merge the request`. Copy **nguyên văn**, không sửa chữ nào ở bước này.

- [ ] **Step 4: Thêm dấu đồng bộ vào 2 block verbatim của Phase 2**

Trong section `review the whole branch`, ngay trước dòng `**Block 1 — Severity:**`, chèn:

```markdown
> 🔗 **Dấu đồng bộ**: Block 1 phải khớp bảng `Review output` ở `gitlab-flow`; dòng lens chép vào mỗi agent phải khớp `Review lenses` ở `gitlab-flow`. Sửa một bên phải sửa bên kia.
```

- [ ] **Step 5: Thêm phần đuôi cho `gitlab-review`**

Cuối file:

```markdown
## Safety rules

Áp dụng toàn bộ `## Safety rules` của `gitlab-flow`, nhấn thêm 2 điều đặc thù vai Reviewer:

- 🚫 **KHÔNG chèn AI attribution** vào comment post lên MR (`Co-Authored-By: Claude`, `🤖 Generated with`, link `claude.com`). Comment = nội dung review thuần.
- **KHÔNG merge** khi thiếu approve / CI fail / có conflict — báo user và hỏi, không tự override.

## Tools required

- `git`
- `glab` (GitLab CLI) — bắt buộc cho cả 4 trigger. Chưa cài: https://gitlab.com/gitlab-org/cli
```

- [ ] **Step 6: Xoá 4 section khỏi `gitlab-flow/SKILL.md`**

Xoá đúng 4 khối đã trích ở Step 1. Giữ nguyên `### "create a merge request"` và `### "fix all issues"`.

- [ ] **Step 7: Sửa mọi cross-reference còn trỏ tới trigger đã dời**

```powershell
Select-String -Path "D:\TemplateAi\Project User Main\skills_end_to_end\skills\gitlab-flow\SKILL.md" -Pattern 'review the whole branch|review the MR|post review result|merge the request' -AllMatches
```

Với **mỗi** match còn lại, thêm chú thích skill. Ba chỗ đã biết:

1. Bảng `Output language` — **xoá 3 dòng** của `review the whole branch`, `review the MR !<N>`, `post review result to the MR` (chúng thuộc `gitlab-review` rồi). Giữ dòng `review the last change` và dòng `fix all issues`.
2. Mục `**Lưu ý — chọn đúng độ sâu**` trong `review the last change` — sửa thành:
   ```markdown
   - Muốn **chính xác/sâu hơn**: dùng skill built-in **`/code-review`**, hoặc trigger **`review the whole branch`** (skill **`gitlab-review`** — chỉ Lead cài; 4 agent chuyên biệt đọc full file + tầng verify).
   - Diff lớn (>500 dòng) hoặc nhiều commit → nhờ Lead chạy `review the whole branch`, đừng cố review inline.
   ```
3. `Commit and push` Step 6 mục 5 (gợi ý bước tiếp) — sửa thành:
   ```markdown
   5. Sau khi push thành công: báo URL push + gợi ý bước tiếp (`create a merge request`; nếu branch nhiều commit thì nhờ Lead chạy `review the whole branch` trước)
   ```

- [ ] **Step 8: Verify frontmatter hợp lệ và không mất nội dung**

```powershell
$new = "D:\TemplateAi\Project User Main\skills_end_to_end\skills\gitlab-review\SKILL.md"
Get-Content $new -TotalCount 4
"desc chars: " + ((Get-Content $new -TotalCount 3)[2]).Length
```

Kỳ vọng: dòng 1 và 4 là `---`; `name: gitlab-review`; description **< 1024 ký tự**.

```powershell
Select-String -Path $new -Pattern '^### "' | Select-Object LineNumber, Line
```

Kỳ vọng: đúng **4** section.

- [ ] **Step 9: Verify tổng từ không hụt**

Chạy `do-tu.ps1` cho cả hai file. Kỳ vọng `gitlab-flow` ≈ 6.550 từ (9.221 − 2.859 + 400 lens − ~200 dòng bị xoá khỏi Output language), `gitlab-review` ≈ 3.100 từ. Tổng ≥ 9.400 (tăng vì thêm header REQUIRES). Tổng **giảm** ⇒ mất nội dung, DỪNG và soát lại.

- [ ] **Step 10: Commit**

```bash
cd "/d/TemplateAi/Project User Main/skills_end_to_end"
git add skills/gitlab-review/SKILL.md skills/gitlab-flow/SKILL.md
git commit -m "$(cat <<'EOF'
feat(gitlab-review): tách 4 trigger vai Reviewer thành skill add-on

Lead chạy toàn bộ review, member chỉ code và commit — nhưng member vẫn
nạp 2.859 từ (31%) của trigger reviewer mỗi lần trigger gitlab-flow.
Member dùng Pro cap thấp nên chịu chi phí này nặng nhất.

gitlab-review tham chiếu Review lenses / Review output / Base branch /
Output language từ gitlab-flow thay vì chép lại, nên không có bản sao
nào để drift. Kèm header REQUIRES chặn trường hợp cài thiếu, vì thiếu
sẽ hỏng ngầm chứ không báo lỗi.
EOF
)"
```

---

### Task 4: Tách `commit-reference.md`

**Files:**
- Create: `skills/gitlab-flow/commit-reference.md`
- Modify: `skills/gitlab-flow/SKILL.md` (mục `### "Commit and push"`)

**Interfaces:**
- Consumes: mục `### "Commit and push"` ở `gitlab-flow`
- Produces: `commit-reference.md` với 6 section `## Allowed types`, `## Footer`, `## Scope`, `## Quick mode`, `## WIP / Spike`, `## Examples`. `SKILL.md` trỏ tới bằng đường dẫn tương đối `commit-reference.md`.

- [ ] **Step 1: Tạo `skills/gitlab-flow/commit-reference.md`**

Header file:

```markdown
# Commit reference

Bảng tra cứu cho trigger `commit and push` của `gitlab-flow`. **Đọc file này khi phân vân**, không cần đọc mỗi lần commit — format và các bước bắt buộc đã nằm inline trong `SKILL.md`.

Mở file này khi: không chắc chọn `type` nào · cần footer `Closes`/`Refs`/`BREAKING CHANGE` · phân vân đặt `scope` · làm commit WIP/Spike · làm commit revert.
```

Rồi **cắt nguyên văn** 6 mục sau từ `SKILL.md` dán vào (đổi cấp heading `####` → `##`):

1. `#### Allowed types` (bảng 11 type + version bump + ghi chú breaking change)
2. `#### Footer` (bảng footer + bảng rule)
3. `#### Scope` (lookup, bảng convention, bảng quyết định new scope, format `.commit-scopes`)
4. `#### Quick mode` (bảng aspect + use for / don't use for)
5. `#### WIP / Spike` (2 bảng + ví dụ)
6. `#### Examples` (5 ví dụ + bảng revert element)

- [ ] **Step 2: Trong `SKILL.md`, thay 6 mục đã cắt bằng khối con trỏ**

Đặt ngay sau `**Step 6 — Push gate**`:

```markdown
#### Tra cứu chi tiết → [`commit-reference.md`](commit-reference.md)

Các bảng sau nằm ở file reference, **đọc khi phân vân**:

| Cần gì | Mục trong `commit-reference.md` |
|---|---|
| Ý nghĩa từng `type` + version bump | `## Allowed types` |
| `Closes` / `Refs` / `BREAKING CHANGE` | `## Footer` |
| Đặt `scope` sao cho đúng, file `.commit-scopes` | `## Scope` |
| Commit `--quick` | `## Quick mode` |
| Commit WIP / Spike | `## WIP / Spike` |
| Ví dụ đầy đủ, format `revert` | `## Examples` |

**11 `type` hợp lệ** (đủ để chọn mà không cần mở file): `feat` · `fix` · `perf` · `refactor` · `docs` · `test` · `build` · `style` · `chore` · `ci` · `revert`.

> Phân vân giữa 2 type (vd dep bump là `build` hay `chore`) ⇒ **mở file reference**, đừng đoán.
```

- [ ] **Step 3: Verify không còn mục nào bị bỏ quên hoặc trùng**

```powershell
$d = "D:\TemplateAi\Project User Main\skills_end_to_end\skills\gitlab-flow"
"--- SKILL.md ---"
Select-String -Path "$d\SKILL.md" -Pattern '^#### (Allowed types|Footer|Scope|Quick mode|WIP|Examples)'
"--- commit-reference.md ---"
Select-String -Path "$d\commit-reference.md" -Pattern '^## '
```

Kỳ vọng: khối đầu **không match** (đã dời hết); khối sau ra đúng **6** section.

- [ ] **Step 4: Verify các rule discipline VẪN ở inline**

```powershell
Select-String -Path "D:\TemplateAi\Project User Main\skills_end_to_end\skills\gitlab-flow\SKILL.md" -Pattern 'Co-Authored-By|Partial-staging guard|Atomic check|Push gate'
```

Kỳ vọng: **có match cho cả 4**. Thiếu bất kỳ cái nào ⇒ đã đẩy nhầm discipline rule ra file phụ, DỪNG và kéo lại.

- [ ] **Step 5: Đo lại**

Chạy `do-tu.ps1` cho `SKILL.md`. Kỳ vọng mục `### "Commit and push"` giảm từ ~1.980 xuống **≤ 800 từ**.

- [ ] **Step 6: Commit**

```bash
cd "/d/TemplateAi/Project User Main/skills_end_to_end"
git add skills/gitlab-flow/commit-reference.md skills/gitlab-flow/SKILL.md
git commit -m "$(cat <<'EOF'
refactor(gitlab-flow): tách bảng tra cứu commit ra commit-reference.md

Commit and push là khối lớn nhất file (1.980 từ) và member luôn phải
nạp, dù ~2/3 là bảng tra cứu chỉ cần khi phân vân.

Ranh giới cắt theo bán kính sát thương nếu Claude không mở file phụ:
thứ cần để làm đúng thì ở lại inline (probe, partial-staging guard,
atomic check, format, self-check no-AI-trailer, push gate), thứ tra khi
phân vân thì ra ngoài. Giữ danh sách 11 tên type inline để chọn được
mà không cần mở file. Discipline rule không bao giờ ra file phụ.
EOF
)"
```

---

### Task 5: `review-branch` → con trỏ

**Files:**
- Modify: `skills/review-branch/SKILL.md` (thay toàn bộ nội dung)

**Interfaces:**
- Consumes: skill `gitlab-review` (Task 3)
- Produces: không có gì cho task sau. Task 6 nhắc file này trong bảng README.

- [ ] **Step 1: Thay toàn bộ nội dung file**

⚠️ **Tiếng Anh** — theo Global Constraints (giữ ngôn ngữ gốc của file) và theo tiền lệ `skills/commit/SKILL.md`, con trỏ deprecated cũng viết tiếng Anh.

```markdown
---
name: review-branch
description: Deprecated pointer. Use when someone invokes review-branch — the whole-branch review workflow now lives in the gitlab-review skill.
---

# review-branch — moved into `gitlab-review`

> ⚠️ **Deprecated as a standalone skill.** The whole-branch review workflow that used to live here now lives in **one canonical place**: the `gitlab-review` skill, under the **`review the whole branch`** trigger. Same move the `commit` skill made into `gitlab-flow`.

Keeping two copies caused real drift: this file went without a severity model entirely while `gitlab-flow` had carried one since commit `733261d`, and a single fix later had to be applied twice, in two languages. One canonical home is the fix.

## Use this instead

```bash
npx skills add nguyenvanchiens/skills_end_to_end -s gitlab-flow -y -a claude-code --copy
npx skills add nguyenvanchiens/skills_end_to_end -s gitlab-review -y -a claude-code --copy
```

`gitlab-review` **depends on** `gitlab-flow` — install both. It reads the severity model, review lenses, base-branch rules, and output-language rules from `gitlab-flow` rather than copying them.

Then type `review the whole branch`. The workflow is unchanged: four specialized agents in parallel → dedup + verify → auto-fix the `Blocker`/`Major` findings that survive verification.

## Not using GitLab / `glab`?

`review the whole branch` only needs `git`. `glab` appears in one optional branch of base-branch detection, which you can skip. The other three `gitlab-review` triggers (`review the MR !N`, `post review result to the MR`, `merge the request`) do need `glab` — don't type them if you don't use GitLab.

## If you want it gone entirely

This file is a deliberate pointer, not the spec. To remove the standalone skill completely, delete the `skills/review-branch/` directory and drop its row from the README — nothing depends on it.
```

- [ ] **Step 2: Verify frontmatter và độ dài**

```powershell
$f = "D:\TemplateAi\Project User Main\skills_end_to_end\skills\review-branch\SKILL.md"
Get-Content $f -TotalCount 4
"tong dong: " + (Get-Content $f).Count
```

Kỳ vọng: frontmatter hợp lệ, tổng **< 40 dòng** (trước đó ~150).

- [ ] **Step 3: Commit**

```bash
cd "/d/TemplateAi/Project User Main/skills_end_to_end"
git add skills/review-branch/SKILL.md
git commit -m "$(cat <<'EOF'
refactor(review): rút review-branch thành con trỏ tới gitlab-review

Bản sao thứ ba của quy trình review branch. Đã drift thật: file này
thiếu hẳn severity model trong khi gitlab-flow có từ commit 733261d,
và hôm nay phải sửa cùng một lỗi hai lần bằng hai ngôn ngữ.

Theo đúng cách skill commit đã deprecated.
EOF
)"
```

---

### Task 6: README — đóng gói 2 bộ + nhận hygiene

**Files:**
- Modify: `README.md`
- Modify: `skills/gitlab-flow/SKILL.md` (xoá `## Skill installation hygiene`)

**Interfaces:**
- Consumes: `gitlab-review` (Task 3), con trỏ `review-branch` (Task 5)
- Produces: không

- [ ] **Step 1: Cắt `## Skill installation hygiene` khỏi `gitlab-flow/SKILL.md`**

Xoá nguyên section (~23 dòng, 211 từ) từ `## Skill installation hygiene` tới hết file.

- [ ] **Step 2: Dán vào cuối `README.md`**

```markdown
## Skill installation hygiene

> Rule cho Claude khi diagnose/fix vấn đề skill (stale, missing behavior, sync issue) — không liên quan workflow GitLab.

🚫 **KHÔNG tự copy/sync skill file vào `C:\Users\<user>\.claude\skills\` (global) trừ khi user yêu cầu rõ ràng.** Cùng nguyên tắc cho mọi system-level location: `~/.claude/`, `%APPDATA%/Claude/`.

**Default khi user báo skill bị stale/sai**:
1. Verify trong repo `skills_end_to_end` đã có version đúng
2. Gợi ý user chạy `npx skills update` trong project bị ảnh hưởng (KHÔNG `-g`)
3. Gợi ý restart Claude session để load skill mới
4. **Chỉ copy thủ công tới global NẾU user yêu cầu rõ** (vd "sync luôn global đi")

**Lý do**: dual-location dễ tạo state lệch nhau; project-only là single source of truth; user có quyền chọn nơi cài, auto-touch global bypass quyền đó.
```

- [ ] **Step 3: Thay mục Cài đặt bằng 2 bộ**

```markdown
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
```

- [ ] **Step 4: Cập nhật bảng skill**

Thêm dòng `gitlab-review` sau `gitlab-flow`:

```markdown
| [`gitlab-review`](skills/gitlab-review/) | **Add-on cho Lead** — vai Reviewer: `review the whole branch` (4 agent chuyên biệt + dedup/verify), `review the MR !N` (mức Grounded, đọc full file tại revision MR mà không cần checkout), `post review result to the MR`, `merge the request`. **Yêu cầu cài kèm `gitlab-flow`** — tham chiếu conventions của nó, không chép lại. | MIT |
```

Sửa dòng `gitlab-flow` thành: `... Quy trình vai Developer: Jira → branch → code → review nhẹ → commit → MR → fix. Phần vai Reviewer tách sang gitlab-review. ...`

Sửa dòng `review-branch` thành: `⚠️ **Deprecated** — gộp vào gitlab-review. File chỉ còn là con trỏ.`

- [ ] **Step 5: Cập nhật mục Recommendation**

```markdown
> **Recommendation**:
> - **Thành viên team**: chỉ cài `gitlab-flow`. Bộ này ~5.250 từ; cài thêm phần Reviewer sẽ đội lên ~8.150 từ mỗi lần trigger mà bạn không dùng tới.
> - **Lead/Maintainer**: `gitlab-flow` + `gitlab-review`, thêm `gitlab-sync` / `gitlab-cherrypick` tuỳ nhu cầu.
> - `commit` và `review-branch` đã deprecated (gộp vào `gitlab-flow` / `gitlab-review`).
```

- [ ] **Step 6: Sửa mục phân vai token**

Trong bảng "Vai / Plan / Làm gì khi review", sửa ô của Member thành: `**KHÔNG** cài gitlab-review (4 agent — đốt token nhiều, và Lead đã lo phần này)`.

- [ ] **Step 7: Verify không còn lệnh cài trỏ sai**

```powershell
Select-String -Path "D:\TemplateAi\Project User Main\skills_end_to_end\README.md" -Pattern '-s review-branch|-s commit\b'
```

Kỳ vọng: **không match** (không còn hướng dẫn cài 2 skill deprecated).

```powershell
Select-String -Path "D:\TemplateAi\Project User Main\skills_end_to_end\skills\gitlab-flow\SKILL.md" -Pattern 'Skill installation hygiene'
```

Kỳ vọng: **không match**.

- [ ] **Step 8: Commit**

```bash
cd "/d/TemplateAi/Project User Main/skills_end_to_end"
git add README.md skills/gitlab-flow/SKILL.md
git commit -m "$(cat <<'EOF'
docs(readme): đóng gói 2 bộ Dev/Lead và nhận mục installation hygiene

Chia lệnh cài thành bộ Dev (1 lệnh) và bộ Lead (4 lệnh) để nói với team
gọn hơn, kèm cảnh báo gitlab-flow là bắt buộc trong bộ Lead.

Mục Skill installation hygiene nói về cách cài/sync skill chứ không phải
workflow GitLab, không lý do gì nạp vào context mỗi lần commit.
EOF
)"
```

---

### Task 7: TESTS.md — T9 / T10 / T11

**Files:**
- Modify: `skills/TESTS.md`

**Interfaces:**
- Consumes: mọi thay đổi từ Task 2-6
- Produces: không

- [ ] **Step 1: Chèn 3 scenario trước mục `## Checklist khi 1 rule fail GREEN`**

```markdown
## T9 — Ranh giới vai (member không được nhận trigger Reviewer)

**Áp dụng:** tách `gitlab-flow` / `gitlab-review`.

**Setup:** môi trường **chỉ** cài `gitlab-flow`, KHÔNG cài `gitlab-review`.

| Rep | Prompt |
|---|---|
| 1 | "review the MR !12" |
| 2 | "review the whole branch" |
| 3 | "merge the request" |

- **GREEN:** Claude nói rõ **không có skill nào phụ trách trigger đó** và gợi ý cài `gitlab-review` (hoặc nhờ Lead). KHÔNG tự bịa quy trình review.
- **Bẫy cần bắt:** `gitlab-flow` vẫn nhắc tên các trigger này ở phần cross-reference. Claude dễ tưởng mình có đủ hướng dẫn rồi tự chế ra 4 agent. Rep 2 là rep nguy hiểm nhất.
- **Cách chấm:** output có mô tả các bước Phase 1/2/2.5 ⇒ **FAIL** (nó đang bịa).

## T10 — Reference nạp theo nhu cầu

**Áp dụng:** `gitlab-flow` `commit and push` + `commit-reference.md`.

| Rep | Prompt | Kỳ vọng GREEN |
|---|---|---|
| 1 | Commit một dep bump (`package.json` đổi version 1 package) | Chọn `build`, **không** `chore` — đúng ghi chú trong `## Allowed types` |
| 2 | Commit revert một commit cũ | Format revert đúng: `This reverts commit <SHA>.` + subject giữ đúng 1 `(TASK-ID)` |
| 3 | Commit đụng 3 module không liên quan | Atomic check STOP hỏi split/combine — rule này **inline**, phải đúng kể cả khi không mở file phụ |
| 4 | Commit thường, 1 file, rõ ràng | **KHÔNG** mở `commit-reference.md` — không cần thì đừng tốn |

- **Cách chấm:** đọc tool-call log. Rep 1-2 phải có `Read` trên `commit-reference.md`; rep 4 phải **không** có.
- **Ý nghĩa:** rep 1-2 fail ⇒ con trỏ trong `SKILL.md` chưa đủ mạnh để Claude biết cần mở. Rep 4 fail ⇒ con trỏ quá mạnh, đang kéo file phụ vào mọi commit và mất hết lợi ích tách file.

## T11 — REQUIRES guard (cài thiếu phải DỪNG)

**Áp dụng:** header của `gitlab-review`.

**Setup:** cài **chỉ** `gitlab-review`, gỡ `gitlab-flow`.

| Rep | Prompt |
|---|---|
| 1 | "review the whole branch" |
| 2 | "review the MR !12" |

- **GREEN:** DỪNG ngay, báo thiếu `gitlab-flow` kèm lệnh cài. **KHÔNG** chạy tiếp.
- **Bẫy chính:** Claude biết thừa severity model là gì (Blocker/Major/Minor/Nit là khái niệm phổ thông) nên rất dễ tự suy ra rồi chạy tiếp. Đó chính là failure mode — review sẽ sai chuẩn mà **không ai phát hiện** vì output trông vẫn bình thường.
- **Cách chấm:** output có bảng severity hay danh sách lens ⇒ **FAIL**, dù nội dung có vẻ đúng.
```

- [ ] **Step 2: Verify**

```powershell
Select-String -Path "D:\TemplateAi\Project User Main\skills_end_to_end\skills\TESTS.md" -Pattern '^## T\d'
```

Kỳ vọng: T1 → T11, đúng **11** match, không trùng số.

- [ ] **Step 3: Commit**

```bash
cd "/d/TemplateAi/Project User Main/skills_end_to_end"
git add skills/TESTS.md
git commit -m "$(cat <<'EOF'
test: thêm T9/T10/T11 cho việc tách skill theo vai

T9 bắt trường hợp member chỉ cài bộ Dev mà Claude tự bịa quy trình
review. T10 kiểm commit-reference.md được mở khi cần và không bị kéo
vào mọi commit. T11 bắt trường hợp cài thiếu gitlab-flow mà Claude tự
suy ra severity model rồi chạy tiếp — failure mode nguy hiểm nhất vì
output trông vẫn bình thường.
EOF
)"
```

---

### Task 8: Verification cuối + đo kết quả

**Files:** không sửa file nào. Chỉ kiểm và báo cáo.

**Interfaces:**
- Consumes: toàn bộ Task 1-7
- Produces: số đo thực tế để đối chiếu với spec

- [ ] **Step 1: Frontmatter của cả 5 skill hợp lệ**

```powershell
$root = "D:\TemplateAi\Project User Main\skills_end_to_end\skills"
Get-ChildItem -Recurse -Filter SKILL.md $root | ForEach-Object {
  $h = Get-Content $_.FullName -TotalCount 4
  [pscustomobject]@{
    Skill  = $_.Directory.Name
    Open   = ($h[0] -eq '---')
    Name   = ($h[1] -match '^name: ')
    Desc   = ($h[2] -match '^description: ')
    DescLen = $h[2].Length
  }
}
```

Kỳ vọng: 5 dòng, `Open`/`Name`/`Desc` đều `True`, `DescLen` < 1024.

- [ ] **Step 2: Không còn tham chiếu gãy trong `gitlab-flow`**

```powershell
$f = "D:\TemplateAi\Project User Main\skills_end_to_end\skills\gitlab-flow\SKILL.md"
"--- section da doi, phai kem ten skill gitlab-review ---"
Select-String -Path $f -Pattern 'review the whole branch|review the MR !|post review result|merge the request'
"--- tro vao bang da doi (phai rong) ---"
Select-String -Path $f -Pattern 'bảng Phase 2|Skill installation hygiene|#### Allowed types|#### Footer|#### Scope|#### Quick mode|#### WIP|#### Examples'
```

Khối 1: mọi match phải có kèm chữ `gitlab-review`. Khối 2: **rỗng**.

- [ ] **Step 3: Discipline rule vẫn inline ở đúng chỗ**

```powershell
$d = "D:\TemplateAi\Project User Main\skills_end_to_end\skills"
"gitlab-flow  no-AI-trailer : " + (Select-String -Path "$d\gitlab-flow\SKILL.md" -Pattern 'Co-Authored-By').Count
"gitlab-flow  severity      : " + (Select-String -Path "$d\gitlab-flow\SKILL.md" -Pattern '^### Review output').Count
"gitlab-flow  lenses        : " + (Select-String -Path "$d\gitlab-flow\SKILL.md" -Pattern '^### Review lenses').Count
"gitlab-review REQUIRES     : " + (Select-String -Path "$d\gitlab-review\SKILL.md" -Pattern 'REQUIRES').Count
```

Kỳ vọng: no-AI-trailer ≥ 3, severity = 1, lenses = 1, REQUIRES ≥ 1.

- [ ] **Step 4: Đo kết quả thật, đối chiếu spec**

Chạy `do-tu.ps1` cho `gitlab-flow/SKILL.md` và `gitlab-review/SKILL.md`, đếm `commit-reference.md`.

| Chỉ số | Dự kiến | Chấp nhận |
|---|---|---|
| Member (`gitlab-flow`) | ~5.050 | 4.800 – 5.400 |
| Lead (2 skill) | ~8.160 | 7.700 – 8.600 |
| `commit-reference.md` | ~1.300 | 1.100 – 1.500 |

Phép tính: `9.221 − 2.859` (trigger Lead) `+ 400` (danh mục lens) `− 200` (3 dòng reviewer bị xoá khỏi bảng `Output language`) `− 1.300` (commit reference) `− 211` (hygiene) `= 5.051`.

> Spec ghi ~5.250 vì lúc đó chưa tính khoản `− 200` của bảng `Output language`. Con số thật thấp hơn spec một chút — **tốt hơn dự kiến, không phải sai**. Không cần sửa spec.

Lệch ngoài khoảng ⇒ báo user kèm số thật, **không tự sửa nội dung để ép cho khớp**.

- [ ] **Step 5: Lịch sử commit sạch**

```bash
cd "/d/TemplateAi/Project User Main/skills_end_to_end"
git log --oneline -8
git status --short
git log -8 --format=%B | grep -iE "co-authored-by|generated with|claude\.com" || echo "OK: khong co AI attribution"
```

Kỳ vọng: 7 commit của Task 1-7 + commit spec; working tree sạch; dòng `OK`.

- [ ] **Step 6: Báo cáo cho user**

Bảng số đo thật vs spec, danh sách commit, và **nhắc rõ**: T9/T10/T11 mới chỉ là test plan, chưa chạy — cần môi trường cài thiếu skill có chủ đích để chạy, và cần user cho phép spawn subagent.

---

## Sau khi xong

Việc còn lại, **không** thuộc plan này:

1. **Chạy T9/T10/T11 + T1 + T7.** Cần user cho phép dùng Agent tool.
2. **Thông báo team cài lại.** Member chạy `npx skills update`; Lead thêm `-s gitlab-review`.
3. **Hướng C (nén nội dung)** — cắt thêm ~15%, làm sau khi cấu trúc mới chạy ổn.
4. **`server/review-runner.ps1`** ở repo khác (`C:\Users\admin\promote-dashboard`, không phải repo này) — câu "đọc thêm code liên quan trong repo" ở dòng 14 giờ có thể đẩy agent về working tree thay vì `FETCH_HEAD`. Đúng bẫy T8 rep 2.
