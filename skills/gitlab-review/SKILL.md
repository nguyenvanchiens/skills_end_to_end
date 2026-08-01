---
name: gitlab-review
description: Use when reviewing a GitLab merge request or a whole feature branch as the reviewer, posting review results back to the MR, or merging an approved MR. Triggers include "review the whole branch", "review the MR !N", "post review result to the MR", "merge the request".
---

# GitLab Review (vai Reviewer / Lead)

> ## ⚠️ REQUIRES `gitlab-flow`
>
> Skill này là **add-on**. Nó tham chiếu `Review lenses`, `Review output` (bảng severity), `Base branch`, `Output language`, `Safety rules`, và **Step 1-2-3 của `review the last change`** (nạp context · 6 tiêu chí review · lọc false positive — đây là rubric đầy đủ cho `review the MR !<N>` mode A) từ skill **`gitlab-flow`** — và **không** chép lại chúng.
>
> ℹ️ **Các mục borrowed KHÔNG có sẵn trong context là chuyện bình thường** — trigger của skill này chỉ nạp file này, không nạp body của `gitlab-flow`. Đó là tín hiệu "đi đọc", **không phải** tín hiệu cài thiếu. Đừng dừng vì lý do này.
>
> Khi cần một mục borrowed, theo đúng thứ tự:
>
> 1. **Đọc `../gitlab-flow/SKILL.md`** (đường dẫn tương đối từ thư mục skill này) và lấy đúng mục đó ra. Chỉ đọc khi thực sự cần — `merge the request` không mượn gì nên không cần đọc.
> 2. **Không tìm thấy file, hoặc file có nhưng thiếu mục đang cần** ⇒ **DỪNG**, báo user cài `gitlab-flow` trước:
>    ```bash
>    npx skills add nguyenvanchiens/skills_end_to_end -s gitlab-flow -y -a claude-code --copy
>    ```
> 3. 🚫 **Tuyệt đối không tự suy ra nội dung thiếu.** Bịa lại bảng severity hay danh mục lens sẽ cho ra review sai chuẩn mà không ai phát hiện.

**Output language**: mặc định tiếng Việt cho cả 4 trigger ở đây, kể cả khi user gõ trigger bằng tiếng Anh — theo mục `Output language` ở `gitlab-flow`.

**Severity**: mọi finding phải gán `Blocker` / `Major` / `Minor` / `Nit` theo bảng `Review output` ở `gitlab-flow`. Danh sách rỗng là kết quả hợp lệ.

## Triggers & Procedures

### "review the whole branch" (review cumulative trước khi mở MR)

Review TOÀN BỘ thay đổi của branch hiện tại so với `<base>` (cách xác định ở Phase 1 — KHÔNG mặc định `main`) — committed + uncommitted — qua 4 agent chuyên biệt song song, verify findings, rồi tự fix. Khác `review the last change` ở điểm: nhìn cumulative diff (nhiều commit), 4 góc nhìn chuyên sâu, có tầng verify, auto-fix các issue đã xác minh.

**Khi nào dùng**: sau khi đã có nhiều commit và push chính, **trước khi `create a merge request`**. Output có thể tạo thêm changes → cần thêm 1 lượt `commit and push` nữa rồi mới mở MR. Bỏ qua bước này nếu branch chỉ 1 commit nhỏ — `review the last change` là đủ.

**Phase 1 — Identify changes**:

1. **Xác định `<base>` — KHÔNG mặc định `main`** (thứ tự ưu tiên, dừng ở match đầu):
   - **Nhánh hiện tại có MR?** Chạy `glab mr view` (không tham số → MR của nhánh đang đứng; áp dụng khi bạn đã checkout nhánh MR bằng glab/git/**SourceTree**). Có MR → dùng `targetBranch` của nó làm `<base>`, **không hỏi** (dự án main-chính → `main`, dev-chính → `dev`). Rỗng/lỗi → bỏ qua.
   - **User nói rõ base trong prompt** ("vs dev") → dùng.
   - **`<base>` đã chốt trong session** (vd lúc `create branch from task`) → tái dùng.
   - **Còn lại** → HỎI `main`/`dev` (xem mục **Base branch** ở `gitlab-flow`).

   Rồi resolve merge base: `git merge-base <base> HEAD`
2. Nếu branch hiện tại IS `<base>` (hoặc base = HEAD) → báo "không có gì để review" và STOP
3. Capture cumulative diff (commit + working tree) vào temp file để các agent đọc mà không flood context. Ghi vào `.git/` (luôn tồn tại, writable, không bị commit) — **KHÔNG dùng `/tmp/`** vì trên Windows/PowerShell `/tmp/` không tồn tại ổn định:
   ```bash
   BASE=$(git merge-base <base> HEAD)
   git diff --no-color "$BASE" > .git/review_branch.diff
   wc -l .git/review_branch.diff
   ```
4. Capture danh sách file untracked (diff không bao gồm):
   ```bash
   git ls-files --others --exclude-standard > .git/review_branch_new.txt
   ```
5. Stat tóm tắt để spot-check:
   ```bash
   git diff --stat "$BASE"
   ```
6. **Nạp task context** — không có bước này thì agent Correctness ở Phase 2 vô nghĩa (không có chuẩn nào để đối chiếu "đúng/sai"):
   - Extract TASK-ID từ tên branch (pattern `[A-Z][A-Z0-9]+-\d+`)
   - Lấy mô tả task, thứ tự ưu tiên dừng ở match đầu: mô tả đã có trong hội thoại → description của MR nếu branch đã mở MR (`glab mr view`) → **HỎI user 1 câu ngắn**: "Task `<TASK-ID>` yêu cầu gì? (1-2 câu là đủ)"
   - Không lấy được (user bỏ qua) → vẫn chạy Phase 2, nhưng **nói rõ với user**: "thiếu task context — agent Correctness chỉ xét self-consistency, không phán được logic có khớp yêu cầu hay không"

**Phase 2 — Launch 4 agent SONG SONG** (1 message, 4 Agent tool calls):

Mỗi agent nhận: đường dẫn diff (`.git/review_branch.diff`) + đường dẫn new-files (`.git/review_branch_new.txt`) + context "cumulative diff branch <name> against <base>" + mô tả task từ Phase 1 bước 6.

⚠️ **Bắt buộc chép nguyên văn CẢ HAI block dưới vào prompt của cả 4 agent** — subagent chạy context riêng, KHÔNG thấy mục Review output ở `Conventions` của `gitlab-flow`, KHÔNG thấy Step 1 của `review the last change` (cũng ở `gitlab-flow`):

> 🔗 **Dấu đồng bộ** — 3 cặp phải khớp nhau, sửa một bên phải sửa bên kia:
> - Block 1 ↔ bảng `Review output` ở `gitlab-flow`
> - dòng lens chép vào mỗi agent ↔ `Review lenses` ở `gitlab-flow`
> - Block 2 (Grounding) ↔ **Step 1 của `review the last change`** ở `gitlab-flow` — overlap phần đọc full file + học convention thật trước khi flag, **không phải cùng 1 quy tắc**: Block 2 có thêm bước grep caller mà Step 1 không có, Step 1 có thêm Grounding task (đọc mô tả Jira) mà Block 2 không có. Sửa phần chung thì sửa cả hai, đừng gộp làm 1

**Block 1 — Severity:**

> Gán severity cho mọi finding: `Blocker` (sai logic/security/mất data/crash) · `Major` (edge case thật, N+1 hot path, race) · `Minor` (naming, code thừa) · `Nit` (style — bỏ, đừng báo). **Đổi behavior của hàm/API dùng chung** (signature giữ nguyên nên compiler không bắt được) mà có ≥1 caller **ngoài diff** bị ảnh hưởng ⇒ **`Blocker`**. Mỗi finding phải có `file:line` + trích code chứng minh, không suy diễn từ diff. **Nếu không tìm thấy Blocker/Major nào, trả về danh sách rỗng — KHÔNG cố tìm cho đủ.**

**Block 2 — Grounding** (đây là thứ tách "review thật" khỏi "đoán từ diff"):

> **Trước khi flag bất cứ điều gì, nạp context — KHÔNG review diff trong ống hút:**
> 1. Với MỌI file xuất hiện trong diff: **đọc FULL file**, không chỉ hunk. Diff cắt mất chính đoạn code quyết định finding đúng hay sai.
> 2. Đọc `CLAUDE.md` (nếu có) + 1-2 file cùng thư mục/module với file đã đổi, để học convention **THẬT** của repo. Đừng áp convention generic.
> 3. Hàm/API/schema bị đổi signature hoặc behavior: `Grep` tìm **mọi caller**, kể cả file KHÔNG nằm trong diff. Thay đổi trông an toàn trong diff vẫn có thể làm gãy caller ở nơi khác — đó là loại lỗi diff-only review không bao giờ thấy.
> 4. Chỉ flag sau khi đã làm 1-3. Finding suy diễn từ diff là nguyên nhân #1 gây false positive.

| Agent | Lens |
|---|---|
| Agent 1 | `Correctness / Task-fit` |
| Agent 2 | `Security` |
| Agent 3 | `Efficiency` |
| Agent 4 | `Quality & Reuse` |

Định nghĩa đầy đủ từng lens (Tập trung · Severity chủ đạo · Flag điển hình): xem **`Review lenses`** ở mục `Conventions` của `gitlab-flow`. **Chép nguyên văn dòng của lens tương ứng** vào prompt agent đó — subagent không thấy `Conventions`.

> **Vì sao roster là 4 agent này**: `Blocker` theo bảng severity = "sai logic vs task, lỗ hổng security, mất data, crash" — nên **Correctness và Security là 2 agent bắt buộc**, chúng phụ trách đúng thứ duy nhất chặn merge. `Quality & Reuse` gộp làm 1 slot vì output của nó gần như toàn `Minor`, mà `Minor` thì không auto-fix — không đáng tách thành 2 agent.

**Phase 2.5 — Dedup + verify (BẮT BUỘC — đừng nối thẳng 4 danh sách rồi fix)**:

4 agent chạy context riêng, mỗi agent chỉ nhìn qua 1 lens → chắc chắn có trùng lặp và có finding sai. Auto-fix ở Phase 3 sẽ **sửa code thật** theo danh sách này, nên sai ở đây đắt hơn sai ở một review chỉ để đọc.

1. **Dedup** theo `file:line` + bản chất vấn đề. Trùng → giữ bản mô tả rõ nhất và ghi nhận có mấy agent cùng báo. **2+ agent độc lập cùng chỉ vào 1 chỗ = tín hiệu mạnh**, xử lý trước.
2. **Verify từng `Blocker`/`Major`** — mặc định là **BÁC BỎ**, chỉ giữ lại khi chứng minh được cả 3:
   - Mở file thật tại `file:line`, đọc code xung quanh → vấn đề có tồn tại ở code hiện tại không, hay agent đọc nhầm hunk?
   - Đường đi tới bug có **reachable** không? (nhánh dead code, caller đã guard, input đã validate ở tầng trên → bác bỏ)
   - Fix đề xuất có áp dụng được với repo này không? (helper/util mà agent gợi ý có tồn tại thật không)
   - Thiếu bất kỳ điều nào → **loại khỏi danh sách auto-fix**. Còn nghi ngờ thì hạ xuống mục "cần xác nhận" cho user, KHÔNG tự sửa.
3. `Minor` chỉ dedup, không cần verify (không auto-fix nên sai cũng không phá code).

> Danh sách sau verify **ngắn hơn nhiều** so với tổng 4 agent. Đó là dấu hiệu ĐÚNG, không phải review kém. Báo cáo số bị loại để user thấy tầng verify có chạy.

**Phase 3 — Fix + báo cáo**:

1. Fix trực tiếp trong working tree **chỉ `Blocker` + `Major` đã qua Phase 2.5**. `Minor` gom vào mục riêng để user tự quyết, `Nit` bỏ. Không còn finding nào sau verify → báo "Không có vấn đề chặn".
2. **KHÔNG tự commit/push** — để user review changes rồi tự `commit and push` (sẽ hỏi xác nhận push như thường lệ)
3. Tóm tắt theo đúng format này — `Minor` phải có chỗ đứng, nếu không item 1 nói "gom vào mục riêng" mà không có mục nào:
   ```
   Đã fix: <N> Blocker, <M> Major — <danh sách file>
   Verify: <T> finding thô → <D> vấn đề sau dedup → bác bỏ <K> → còn <R> để fix
   Test/typecheck: <status>

   ### Minor (không fix — user tự quyết)
   #1 [Minor] path/file.cs:15 — <vấn đề>

   ### Cần xác nhận (không tự fix)
   #2 path/file.cs:80 — <vấn đề>. Chưa chứng minh được vì <lý do>
   ```
4. Gợi ý bước tiếp: nếu có fix → `commit and push` rồi `create a merge request`; nếu không có gì cần sửa → `create a merge request` luôn

**Lưu ý**:
- Diff > 2000 dòng → review có thể coarse-grained. Khuyến cáo user lần sau chạy sớm hơn (sau mỗi vài commit) thay vì để dồn cuối.
- Base branch (`master`/`develop`/`dev` thay `main`) → xem mục **Base branch** ở `gitlab-flow`. Nếu `<base>` đã được chốt lúc tạo branch thì tái dùng; nếu chưa rõ (vd review branch tạo ngoài skill) → hỏi user `main` hay `dev` trước khi `git merge-base`.
- Trigger này chuyên review macro. Để review chỉ thay đổi gần nhất → dùng `review the last change`. Để review MR đã push (vai Reviewer) → dùng `review the MR !<N>`.

### "review the MR !<N>" (vai trò Reviewer)

1. Yêu cầu `glab` CLI đã cài: kiểm tra `glab --version`
2. Lấy thông tin MR + comment đã có:
   - `glab mr view <N> --comments` (hiển thị cả note/discussion đã có)
3. **BẮT BUỘC** lấy diff từ remote bằng `glab mr diff <N>`. **KHÔNG** thay thế bằng `git diff <base>...<source>` so với branch local — `main` (hoặc base) ở local có thể stale, dẫn tới review nhầm hàng trăm commits đã có sẵn trên remote. Nếu thực sự cần dùng `git diff` (vd để lấy stat), phải `git fetch origin <base-branch>` trước rồi so với `origin/<base-branch>`, không phải branch local.

   > **3 mức độ sâu — chọn theo nhu cầu:**
   >
   > | Mức | Context có được | Cần checkout? |
   > |---|---|---|
   > | **Inline** | Chỉ text diff từ `glab mr diff <N>` | Không |
   > | **Grounded** ← *mặc định* | Full file + caller tại đúng revision của MR | **Không** |
   > | **Deep** | Grounded + 4-agent ensemble + verify | Có |
   >
   > **Grounded — đọc full file mà KHÔNG đụng working tree.** Đây là mức mặc định vì nó gỡ được điểm yếu chí mạng của review chỉ-nhìn-diff mà không bắt user rời nhánh đang làm dở:
   > ```bash
   > # <source-branch> lấy từ `glab mr view <N>` (field source branch)
   > git fetch origin <source-branch>
   > git show FETCH_HEAD:path/to/file.js       # full file tại đúng revision của MR
   > git grep -n "functionName" FETCH_HEAD     # tìm caller trong TOÀN repo tại revision đó
   > ```
   > ⚠️ **TUYỆT ĐỐI KHÔNG đọc file ở working tree để lấy context cho MR.** Working tree đang ở nhánh khác — nội dung file có thể khác hoàn toàn với revision của MR. Ghép diff của MR với context của nhánh khác cho ra finding sai mà nghe rất thuyết phục. Chỉ đọc qua `FETCH_HEAD:` (hoặc sau khi đã checkout đúng nhánh MR).
   > ⚠️ Fetch fail (offline / chưa auth) → tụt về mức **Inline** và **nói rõ với user**, đừng im lặng đọc working tree thay thế.
   >
   > **Deep** — chỉ khi cần 4-agent ensemble: về đúng nhánh source của MR trước (checkout bằng `glab mr checkout <N>`, `git checkout`, hoặc **SourceTree/GUI** — tool nào cũng được, skill chỉ cần HEAD ở đúng nhánh), rồi chạy `review the whole branch`, sau đó quay lại bước 4-5 để post. ⚠️ `review the whole branch` / `/code-review` đọc code **local** — không soi được remote MR khi đang đứng ở nhánh khác, nên checkout là bắt buộc nếu muốn deep.
   >
   > **Base cho deep review = target branch của MR**, KHÔNG hỏi `main`/`dev`. Lấy bằng `glab mr view <N>` (field targetBranch) — dự án main-chính ra `main`, dự án dev-chính ra `dev`. Dùng đúng nhánh đó làm `<base>` khi `review the whole branch` (đây là ngữ cảnh reviewer: base đã xác định sẵn từ MR, không thuộc trường hợp "hỏi user" ở Phase 1).
4. **Phân nhánh theo trạng thái comment**:

   **(A) MR CHƯA có comment review nào** → review mới hoàn toàn:
   - **Nạp context trước đã** (mức Grounded — xem bảng 3 mức ở trên): `git fetch origin <source-branch>`, rồi `git show FETCH_HEAD:<file>` để đọc **full file** của mọi file trong diff, và `git grep -n <symbol> FETCH_HEAD` để truy caller của hàm/API bị đổi signature. Cũng đọc `CLAUDE.md` tại `FETCH_HEAD` nếu có.
   - Review toàn bộ diff theo tiêu chí Step 2 của mục "review the last change" (ở `gitlab-flow`). Có full file → **áp nguyên Step 3** ("chứng minh bằng code đã đọc") như bình thường
   - ⚠️ Chỉ khi fetch thất bại và phải tụt về mức **Inline** (chỉ có text diff) thì mới nới Step 3: finding nào cần context ngoài diff ghi **"cần xác nhận"**, đừng khẳng định. Nói rõ trong output là review chạy ở mức Inline
   - Liệt kê issues `#1 [Severity] file:line — <vấn đề>. Đề xuất: <fix>`
   - Verdict map thẳng từ severity: còn `Blocker`/`Major` → `REQUEST_CHANGES` · chỉ còn `Minor` → `COMMENT` · sạch → `APPROVE` (nói rõ "không có vấn đề chặn", không bịa Nit để có cái mà comment)

   **(B) MR ĐÃ có comment review trước đó** → review tiếp nối, KHÔNG review lại từ đầu:
   - Đọc kỹ comment cũ, trích xuất danh sách issue đã raise (`#1`, `#2`, ...) kèm verdict gần nhất
   - Xác định mốc thời gian / commit của lần review trước (lấy `created_at` của note review cuối, hoặc commit SHA mà reviewer reference)
   - Lấy commit mới push từ sau mốc đó: `glab mr view <N>` → xem `commits` hoặc `git log <last-reviewed-sha>..origin/<source-branch>`
   - **Đối chiếu issue cũ**: với mỗi issue `#N` đã raise, kiểm tra trong commit/diff mới xem đã được fix chưa. Đánh dấu:
     - `✓ Resolved #N` — đã fix đúng
     - `❌ Still open #N` — chưa fix, hoặc fix sai/chưa đủ — kèm lý do
     - `⚠️ Partially #N` — fix một phần, kèm điều còn thiếu
   - **Issue mới phát sinh từ commit mới**: đánh số tiếp theo (`#N+1`, `#N+2`, ...), không tái sử dụng số cũ
   - **KHÔNG** review lại các phần code không thay đổi từ lần review trước (trừ khi liên quan trực tiếp tới issue cũ)
   - Verdict: `APPROVE` nếu mọi `Blocker`/`Major` cũ đã `✓ Resolved` và không có `Blocker`/`Major` mới; `REQUEST_CHANGES` nếu còn `Blocker`/`Major` ở trạng thái `❌ Still open` hoặc mới phát sinh; `COMMENT` cho phần còn lại. ⚠️ **`Minor` chưa fix KHÔNG chặn `APPROVE`** — Minor theo thiết kế là không fix trong branch này, nếu tính vào thì MR treo vĩnh viễn
5. Output format thống nhất:
   ```
   ## Review !<N> (lần thứ <K>)

   **Verdict:** APPROVE | REQUEST_CHANGES | COMMENT

   ### Trạng thái issue cũ        ← chỉ có ở mode (B)
   - ✓ Resolved #1
   - ❌ Still open #2 — <lý do>
   - ⚠️ Partially #3 — <còn thiếu>

   ### Issue mới
   - #N+1 [Blocker] `path/to/file.js:42` — <vấn đề>. Đề xuất: <fix>
   - #N+2 [Major]   `path/to/file.js:88` — <vấn đề>. Đề xuất: <fix>

   ### Minor (không chặn merge)
   - #N+3 [Minor]   `path/to/file.js:15` — <vấn đề>
   ```

### "post review result to the MR"

> 🚫 **KHÔNG chèn AI attribution** vào comment (Co-Authored-By Claude, 🤖 Generated with, link claude.com, ...). Comment = chỉ nội dung review thuần, KHÔNG signature/footer.

1. Lấy chính output Markdown từ bước "review the MR" trước đó (đã đúng format, không cần soạn lại). Nếu là review tiếp nối (mode B), giữ nguyên cả phần "Trạng thái issue cũ" — đó là context quan trọng cho dev.
2. **Self-check trước khi `glab mr note`**: scan output Markdown, đảm bảo không có keyword `Claude`, `Anthropic`, `🤖`, `Generated with`, `Co-authored-by:`, `claude.com`. Có → xóa.
3. Đăng comment: `glab mr note <N> --message "<markdown>"`
4. Nếu APPROVE: `glab mr approve <N>`
5. Nếu REQUEST_CHANGES với toàn bộ issue cũ đã `✓ Resolved` (chỉ còn issue mới): nói rõ trong comment để dev biết phần fix trước đã OK

### "merge the request"
1. Kiểm tra MR đã có:
   - At least 1 approve
   - CI pipeline pass: `glab mr view <N>` (hoặc `glab ci status`)
   - Không có conflict
2. Nếu thiếu điều kiện, BÁO CHO USER và hỏi có override không (KHÔNG tự ý merge)
3. Merge: `glab mr merge <N> --remove-source-branch --squash`
4. Checkout về `<base>` = **target branch thật của MR** (lấy từ `glab mr view <N>` field `targetBranch`, không đoán `main`), pull về bản mới nhất
5. Báo merge thành công + commit hash trên `<base>`

## Safety rules

Áp dụng toàn bộ `## Safety rules` của `gitlab-flow`, nhấn thêm 2 điều đặc thù vai Reviewer:

- 🚫 **KHÔNG chèn AI attribution** vào comment post lên MR (`Co-Authored-By: Claude`, `🤖 Generated with`, link `claude.com`). Comment = nội dung review thuần.
- **KHÔNG merge** khi thiếu approve / CI fail / có conflict — báo user và hỏi, không tự override.

## Tools required

- `git`
- `glab` (GitLab CLI) — bắt buộc cho `review the MR !<N>`, `post review result to the MR`, `merge the request`. **`review the whole branch` chỉ cần `git`** — 2 chỗ dùng `glab mr view` ở Phase 1 (xác định `<base>` từ MR đang mở, lấy mô tả task) đều optional, có fallback khi rỗng/lỗi. Chưa cài: https://gitlab.com/gitlab-org/cli
