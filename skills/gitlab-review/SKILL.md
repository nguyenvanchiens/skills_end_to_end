---
name: gitlab-review
description: Use when reviewing a GitLab merge request or a whole feature branch as the reviewer, posting review results back to the MR, or merging an approved MR. Triggers include "review the whole branch", "review the MR !N", "post review result to the MR", "merge the request".
---

# GitLab Review (vai Reviewer / Lead)

> ## ⚠️ REQUIRES `gitlab-flow`
>
> Skill này là **add-on**. Nó tham chiếu `Review lenses` (kèm `Flag kích hoạt theo tình huống` ngay dưới bảng), `Review output` (bảng severity), `Base branch`, `Output language`, `Safety rules`, `Chuỗi mặc định giữa các trigger`, và **Step 1-2-3-4 của `review the last change`** (nạp context kể cả ngoài diff · tiêu chí review · lọc false positive + đường đo · tự soát báo cáo — đây là rubric đầy đủ cho `review the MR !<N>` mode A) từ skill **`gitlab-flow`** — và **không** chép lại chúng.
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

**Chuỗi mặc định** (phần của skill này trong chuỗi khai ở `gitlab-flow`): `create a merge request` → `review the MR !<N>` → **[`post review result to the MR`]** → dev `fix all issues` → **[commit/push]** → `review the MR !<N>` lượt sau → **[`merge the request`]**. Mắt chỉ chạy local (review, đo, fix trong working tree) thì chạy luôn; mọi cổng trong `[...]` — post, commit, push, merge — vẫn bắt buộc hỏi. **Gọi lại một trigger với đầu vào không đổi = lệnh THỰC THI**, không phải yêu cầu kiểm lại tiền đề: mỗi nhánh "hỏi user" chỉ được hỏi MỘT lần.

## Triggers & Procedures

### "review the whole branch" (review cumulative trước khi mở MR)

Review TOÀN BỘ thay đổi của branch hiện tại so với `<base>` (cách xác định ở Phase 1 — KHÔNG mặc định `main`) — committed + uncommitted — qua 4 agent chuyên biệt song song, verify findings, rồi tự fix. Khác `review the last change` ở điểm: nhìn cumulative diff (nhiều commit), 4 góc nhìn chuyên sâu, có tầng verify, auto-fix các issue đã xác minh.

**Khi nào dùng**: sau khi đã có nhiều commit và push chính, **trước khi `create a merge request`**. Output có thể tạo thêm changes → cần thêm 1 lượt `commit and push` nữa rồi mới mở MR. Bỏ qua bước này nếu branch chỉ 1 commit nhỏ — `review the last change` là đủ.

**Phase 1 — Identify changes**:

1. **Xác định `<base>` — KHÔNG mặc định `main`** (thứ tự ưu tiên, dừng ở match đầu):
   - **Nhánh hiện tại có MR?** Chạy `glab mr view` (không tham số → MR của nhánh đang đứng; áp dụng khi bạn đã checkout nhánh MR bằng glab/git/**SourceTree**). Có MR → dùng `targetBranch` của nó làm `<base>`, **không hỏi** (dự án main-chính → `main`, dev-chính → `dev`). Rỗng/lỗi → bỏ qua.
   - **User nói rõ base trong prompt** ("vs dev") → dùng.
   - **`<base>` đã chốt trong session** (vd lúc `create branch from task`) → tái dùng.
   - **Còn lại** → HỎI `main`/`dev` (xem mục **Base branch** ở `gitlab-flow` — nhánh deploy/release không được vào danh sách option).

   Rồi resolve merge base: `git merge-base <base> HEAD`
2. Nếu branch hiện tại CHÍNH LÀ `<base>` → báo "không có gì để review" và STOP. **Chỉ** điều kiện này được suy từ con trỏ commit — `merge-base == HEAD` KHÔNG phải lý do dừng (xem bước 4)
3. Capture cumulative diff (commit + working tree) vào temp file để các agent đọc mà không flood context. Ghi vào `.git/` (luôn tồn tại, writable, không bị commit) — **KHÔNG dùng `/tmp/`** vì trên Windows/PowerShell `/tmp/` không tồn tại ổn định:
   ```bash
   BASE=$(git merge-base <base> HEAD)
   git diff --no-color "$BASE" > .git/review_branch.diff      # khong kem HEAD: co y bao ca working tree
   git ls-files --others --exclude-standard > .git/review_branch_new.txt
   wc -l .git/review_branch.diff .git/review_branch_new.txt
   ```
4. **Chốt "không có gì để review" đo ĐỘ DÀI diff vừa capture**, không đo quan hệ con trỏ: cả hai file cùng rỗng mới STOP. `merge-base == HEAD` chỉ nghĩa là mọi commit đã lên base (bình thường khi review vòng tiếp trên nhánh có MR trước đã merge) — working tree vẫn có thể còn cả trăm dòng. Khi đó nói rõ trong báo cáo: phạm vi lượt này là thay đổi chưa commit
5. Stat tóm tắt để spot-check:
   ```bash
   git diff --stat "$BASE"
   ```
6. **Nạp task context** — không có bước này thì agent Correctness ở Phase 2 vô nghĩa (không có chuẩn nào để đối chiếu "đúng/sai"):
   - Extract TASK-ID từ tên branch (pattern `[A-Z][A-Z0-9]+-\d+`)
   - Lấy mô tả task, thứ tự ưu tiên dừng ở match đầu: mô tả đã có trong hội thoại → description của MR nếu branch đã mở MR (`glab mr view`) → **HỎI user 1 câu ngắn**: "Task `<TASK-ID>` yêu cầu gì? (1-2 câu là đủ)"
   - Không lấy được (user bỏ qua) → vẫn chạy Phase 2, nhưng **nói rõ với user**: "thiếu task context — agent Correctness chỉ xét self-consistency, không phán được logic có khớp yêu cầu hay không"
   - Task/user nêu một repo hoặc file làm **khuôn** ("làm theo bên X", "mirror module Y") → ghi đường dẫn khuôn vào context cho cả 4 agent (Block 2 bước 6 review cả khuôn, không chỉ so cho khớp)

**Phase 2 — Launch 4 agent SONG SONG** (1 message, 4 Agent tool calls, **foreground** — `run_in_background: false`): agent nền có thể trả kết quả về nhầm cha, hoặc bị cắt khi chính phiên này đang là subagent — rồi bước gộp chạy trên 1/4 lens mà không ai biết.

Mỗi agent nhận: đường dẫn diff (`.git/review_branch.diff`) + đường dẫn new-files (`.git/review_branch_new.txt`) + context "cumulative diff branch <name> against <base>" + mô tả task từ Phase 1 bước 6 — **và KHÔNG gì khác**: không chẩn đoán, kết luận, "bản vá đúng hướng", hay tóm tắt diễn biến phiên (agent nhận đáp án sẽ đi xác nhận đáp án, không soi). Mô tả task/MR đi kèm chỉ thị kiểm chứng lại từng khẳng định trong đó, kể cả con số.

⚠️ **Bắt buộc chép nguyên văn CẢ HAI block dưới vào prompt của cả 4 agent** — subagent chạy context riêng, KHÔNG thấy mục Review output ở `Conventions` của `gitlab-flow`, KHÔNG thấy Step 1 của `review the last change` (cũng ở `gitlab-flow`):

> 🔗 **Dấu đồng bộ** — 3 cặp phải khớp nhau, sửa một bên phải sửa bên kia:
> - Block 1 ↔ bảng `Review output` ở `gitlab-flow` (gồm các gạch đầu dòng "cần xác nhận" và "HẠ severity cũng là một khẳng định")
> - dòng lens + nhóm `Flag kích hoạt theo tình huống` của lens đó chép vào mỗi agent ↔ `Review lenses` ở `gitlab-flow`
> - Block 2 (Grounding) ↔ **Step 1 của `review the last change`** ở `gitlab-flow` — bước 3–7 ở đây ↔ bước 5 (a–e) ở đó, bước 8 ↔ khối "Phủ phần CÒN LẠI" ở Step 3 đó. **Không phải cùng 1 quy tắc**: Block 2 có thêm bước 9 (mặc định của framework), Step 1 có thêm Grounding task (đọc mô tả Jira). Sửa phần chung thì sửa cả hai, đừng gộp làm 1
>
> ⚠️ Danh sách dấu đồng bộ này cũng là giả thuyết: một câu của bản gốc từng rơi mất trên đường chép sang Block 1 mà không báo lỗi gì. Mỗi lần sửa, đối chiếu **từng câu** của bản gốc với bản sao, không chỉ tick từng cặp.

**Block 1 — Severity:**

> Gán severity cho mọi finding: `Blocker` (sai logic/security/mất data/crash) · `Major` (edge case thật, N+1 hot path, race) · `Minor` (naming, code thừa) · `Nit` (style — bỏ, đừng báo). **Đổi behavior của hàm/API dùng chung** (signature giữ nguyên nên compiler không bắt được) mà có ≥1 caller **ngoài diff** bị ảnh hưởng ⇒ **`Blocker`**. Mỗi finding phải có `file:line` + trích code chứng minh, không suy diễn từ diff; gán không nổi severity ⇒ chưa đủ rõ.
>
> **Có BA loại đầu ra, không phải hai:** (1) finding có severity; (2) danh sách rỗng — không tìm thấy Blocker/Major thì trả rỗng, **KHÔNG cố tìm cho đủ**; (3) **`Cần xác nhận`** — điều bạn tin là quan trọng nhưng KHÔNG chứng minh được bằng code đã đọc (cần đo trên bản đang chạy, hai bằng chứng ngược nhau, phụ thuộc cách framework lập lịch…). Báo ở mục riêng `### Cần xác nhận`, kèm: nghi ngờ là gì, các bằng chứng đối nghịch, và **phép đo cụ thể nào sẽ phân xử**. 🚫 Không nâng nó lên `Major` để có chỗ đặt — `Major` bị đem đi SỬA CODE THẬT. 🚫 Cũng không im lặng bỏ: nghi ngờ về *tiền đề* của diff thường đáng giá hơn mọi finding trong diff.
>
> **Hạ severity phải có lý do kiểm được:** "hiện trạng repo vốn thế" không hạ được khi chính MR có chỗ làm đúng luật đó (trích chỗ đó); "repo đã có tiền lệ" chỉ hạ được khi tiền lệ là một quyết định có lý do, không phải thứ tích tụ; severity theo hệ quả nặng nhất của DÒNG CODE, không theo lập luận vừa bị bác; hạ bằng phép đo ⇒ đo TỪNG nguồn dữ liệu finding phủ và kiểm tầng chặn phía sau có đang BẬT (cờ `DryRun`/`Enabled`/`Bypass` trong cấu hình của đúng app). Mục "dưới ngưỡng / đã cân nhắc bỏ qua" cũng liệt kê ra kèm lý do — orchestrator sẽ xét lại.

**Block 2 — Grounding** (đây là thứ tách "review thật" khỏi "đoán từ diff"):

> **Trước khi flag bất cứ điều gì, nạp context — KHÔNG review diff trong ống hút:**
> 1. Với MỌI file xuất hiện trong diff: **đọc FULL file**, không chỉ hunk. Diff cắt mất chính đoạn code quyết định finding đúng hay sai.
> 2. Đọc `CLAUDE.md` (nếu có) + 1-2 file cùng thư mục/module với file đã đổi, để học convention **THẬT** của repo. Đừng áp convention generic.
> 3. Hàm/API/schema bị đổi signature hoặc behavior: `Grep` tìm **mọi caller**, kể cả file KHÔNG nằm trong diff. Field/cột bị gỡ khỏi mọi đường GHI mà vẫn còn trong model/DB (diff chỉ có dòng `-`) ⇒ grep **tên field trần** toàn repo tìm bên ĐỌC. Thêm đường gọi / chế độ chạy / entry point mới ⇒ grep tài liệu, chú thích, header file cấu hình mô tả đường cũ như đường duy nhất, kể cả file ngoài diff. Chiều NGƯỢC: giá trị từ request/client mà diff lưu hoặc truyền đi ⇒ lần tới **sink** cuối (filesystem, SQL thô, shell, URL, xoá dữ liệu) kể cả sink ngoài diff, kiểm sink tự validate hay tin caller — sink hổng sẵn mà MR mở đường mới tới ⇒ tính cho MR.
> 4. Kết luận **phủ định** ("không caller", "không có luật") ⇒ quét lần hai rộng hơn hẳn (bỏ mọi bộ lọc đường dẫn) và ghi phạm vi đã quét vào kết luận. Kết quả MỘT PHẦN không phải bằng chứng phép quét đủ rộng.
> 5. Caller ở **repo khác** ⇒ tìm MR/nhánh đồng hành (cùng TASK-ID/tên nhánh: `glab mr list`, rồi `git ls-remote --heads origin | grep <TASK-ID>`), fetch vào ref riêng và đọc ở đó — **không** đọc working tree của repo đó. Đã sửa ở MR đồng hành ⇒ không còn là "quên sửa" mà là ràng buộc thứ tự deploy (`Major`).
> 6. MR dựng từ **một mẫu**: file tự khai "mirror/port/copy từ X" ⇒ diff các **guard** của từng cặp hàm (early-return, `await` trước payload, cờ đang-xử-lý, `?.`/`??`, điều kiện trước `setState`) và đọc chú thích quanh guard ở X làm bằng chứng severity — độ trùng thấp không loại trừ guard bị rơi. Bám **khuôn** từ repo khác ⇒ review cả khuôn ("đang chạy production" không phải "đã kiểm"). Thêm **thành viên thứ N+1** vào họ cùng khuôn ⇒ lấy hiệu tập file với từng anh em; field định danh (port, base URL, AppId, tên DB/queue) trùng anh em là lỗi. Diff với khuôn chỉ thấy cái ĐÃ đổi ⇒ grep trong bản mới mọi định danh đặc thù của khuôn (tên app, cổng, host, đường dẫn, ClientId — cả trong chú thích); mỗi chỗ còn lại phải chứng minh đúng cho bản mới, "giống khuôn" không phải bằng chứng.
> 7. Chú thích **biện minh / viện dẫn** là lời khai: artefact được viện dẫn làm nguồn ⇒ `git ls-tree -r --name-only` / `git grep` tại đúng revision (không có ⇒ so với code tương đương gần nhất đang chạy — "gần" đo bằng HÀNH VI trích từ chính câu viện dẫn, không bằng tên; nguồn NGOÀI repo còn đang sửa ⇒ mở ở trạng thái hôm nay, đồng bộ `gitlab-flow` Step 1 5e); lý do liệt kê N mục đích ⇒ tick với TỪNG mục; "ở đây không cần … như <module khác>" ⇒ đi đọc module đó.
> 8. Phủ phần **còn lại**: MR dọn dẹp ⇒ tự sinh danh sách thực thể cùng loại từ repo rồi phân giải từng cái (đối chiếu lời khai của tác giả là phép kiểm yếu), soi dòng kề dòng đã sửa; nhưng "còn sót chỗ khác" phải đo ở TẦNG bản sửa nằm — ứng viên hội tụ vào leaf đã vá ⇒ loại. "Đã chặn giá trị xấu" ⇒ liệt kê mọi đường RA (render, xuất file, clipboard, URL, log, payload).
> 9. Diff đụng **cấu hình framework** hoặc tầng metadata/SEO/routing ⇒ đọc giá trị MẶC ĐỊNH của thư viện cho mọi tuỳ chọn KHÔNG khai trong code, rồi liệt kê thứ framework tự phát ra ngoài code (HTTP header, cookie, redirect). Tuỳ chọn không khai vẫn là một quyết định — nó chỉ không hiện trong diff.
> 10. Chỉ flag sau khi đã làm 1-9 (bước nào không khớp tình huống thì bỏ qua). Finding suy diễn từ diff là nguyên nhân #1 gây false positive.

| Agent | Lens |
|---|---|
| Agent 1 | `Correctness / Task-fit` |
| Agent 2 | `Security` |
| Agent 3 | `Efficiency` |
| Agent 4 | `Quality & Reuse` |

Định nghĩa đầy đủ từng lens (Tập trung · Severity chủ đạo · Flag điển hình): xem **`Review lenses`** ở mục `Conventions` của `gitlab-flow`. **Chép nguyên văn dòng của lens tương ứng, kèm nhóm của lens đó trong `Flag kích hoạt theo tình huống`** (ngay dưới bảng) vào prompt agent đó — subagent không thấy `Conventions`. Thiếu nhóm flag thì agent chỉ còn danh sách flag chung, và các loại lỗi mà compiler/test/đọc-diff đều không bắt sẽ không có ai hỏi.

> **Vì sao roster là 4 agent này**: `Blocker` theo bảng severity = "sai logic vs task, lỗ hổng security, mất data, crash" — nên **Correctness và Security là 2 agent bắt buộc**, chúng phụ trách đúng thứ duy nhất chặn merge. `Quality & Reuse` gộp làm 1 slot vì output của nó gần như toàn `Minor`, mà `Minor` thì không auto-fix — không đáng tách thành 2 agent.

**Phase 2.5 — Dedup + verify (BẮT BUỘC — đừng nối thẳng 4 danh sách rồi fix)**:

4 agent chạy context riêng, mỗi agent chỉ nhìn qua 1 lens → chắc chắn có trùng lặp và có finding sai. Auto-fix ở Phase 3 sẽ **sửa code thật** theo danh sách này, nên sai ở đây đắt hơn sai ở một review chỉ để đọc.

**Mở đầu bằng phép đếm cứng: "đã nhận N/4 lens".** Danh sách rỗng từ 1/4 lens trông y hệt danh sách rỗng từ 4/4. N<4 ⇒ chạy lại lens thiếu (foreground) rồi mới dedup; buộc phải nộp khi N<4 ⇒ kết luận duy nhất được phép là **"CHƯA ĐỦ — thiếu lens X"** — 🚫 không "Không có vấn đề chặn", không verdict.

1. **Dedup** theo `file:line` + bản chất vấn đề. Trùng → giữ bản mô tả rõ nhất và ghi nhận có mấy agent cùng báo. **2+ agent độc lập cùng chỉ vào 1 chỗ = tín hiệu mạnh**, xử lý trước.
   - ⚠️ **Cùng chỗ nhưng đề xuất LOẠI TRỪ nhau** (cách chữa của lens này kích hoạt đúng lỗi lens kia chỉ ra) ⇒ KHÔNG dedup, cũng không chọn phe. Đó là dấu hiệu cả hai đang chữa triệu chứng: ghi ra bất biến mỗi bên đang bảo vệ, rồi tìm giả định thứ ba khiến hai bất biến xung đột (thường là khoá định danh hoặc NGUỒN của giá trị). Chỉ khi chứng minh được hai bất biến thật sự không cùng thoả được mới chọn — và khi đó trình đánh đổi cho user, không tự chọn.
2. **Verify từng `Blocker`/`Major`** — mặc định **KHÔNG vào auto-fix** cho tới khi chứng minh được cả 5 điều dưới:
   - Mở file thật tại `file:line`, đọc code xung quanh → vấn đề có tồn tại ở code hiện tại không, hay agent đọc nhầm hunk?
   - Đường đi tới bug có **reachable** không? (nhánh dead code, caller đã guard, input đã validate ở tầng trên → bác bỏ)
   - Fix đề xuất có áp dụng được **và ĐÚNG** không? Helper/util agent gợi ý có tồn tại thật không; rồi đọc phần mã BẢN FIX sẽ chạm (hiếm khi trùng chỗ finding chỉ) cùng mọi consumer phía sau: *sau fix, đường đi mới chạy qua những gì?* Fix sinh hình dạng dữ liệu / nhánh trước đây không tới được ⇒ code mới, mặc định chưa an toàn. Fix là **cổng chặn** (validation, equality, guard) ⇒ thử chiều false-positive: ca hợp lệ nào đi qua cổng, đầu vào cổng có luôn sẵn và chuẩn hoá giống nhau ở mọi đường ghi (state nạp bất đồng bộ có thể null; roundtrip đọc-sửa-ghi idempotent?). Fix một **lớp lỗi** ⇒ viết lớp đó thành một câu không chứa tên hàm, grep toàn module, liệt kê MỌI chỗ khớp: đã sửa hay cố ý bỏ. Fix ở tầng **schema** (DDL, migration, script khởi tạo) ⇒ chỉ là "đã sửa" khi chạy trên DB ĐÃ CÓ dữ liệu, hai lần, rồi đo lại trạng thái (`CREATE TABLE IF NOT EXISTS` bỏ cả thân lệnh khi bảng đã có; file đã có khối bù/`ALTER` có điều kiện là lời khai rằng sửa `CREATE` không đủ); SQL dạng chuỗi ⇒ chạy trên đúng engine — build không kiểm nó.
   - **Bằng chứng GIÁN TIẾP là giả thuyết** — phân giải bằng chính thứ nó đứng thay: tên nghe giống (file/hàm đã đổi tên) ⇒ trích tiêu chí HÀNH VI ngay tại chỗ tham chiếu rồi grep theo đó; hàm lân cận cùng class ⇒ truy instance thật gọi hàm (field/factory → datasource → engine) trước khi flag luật riêng của một engine; số đo ("làm đỏ N test", "chậm X%") ⇒ số đó đo phương án NÀO — đo lại đúng phương án đang bàn; triệu chứng khớp giả thuyết quen ⇒ liệt kê nguyên nhân khác cho cùng triệu chứng, chọn phép đo PHÂN BIỆT được; lý do vì sao chỗ cùng dạng lại chạy đúng ⇒ trích từ mã (kể cả mã thư viện), không trích được thì không ghi — lý do bịa sẽ bị chép vào tài liệu như luật.
   - **Finding do nhánh GÂY RA, hay chỉ được lens PHÁT HIỆN?** (tiêu chí phân loại, không phải loại bỏ) Đối chiếu `file:line` với danh sách file trong diff — sink hổng sẵn ngoài diff mà nhánh mở đường mới tới tính là GÂY RA. Trong diff → auto-fix như thường. Ngoài diff → mặc định KHÔNG auto-fix, báo mục riêng; ngoại lệ duy nhất: thay đổi cơ học, cùng loại y hệt việc nhánh đang làm, ở chỗ nhánh bỏ sót, đầu ra không đổi và có test/typecheck khoá. Tái cấu trúc, đổi thứ tự lời gọi, đụng chốt bảo mật: không bao giờ tự sửa dù verify xanh.
   - Thiếu bất kỳ điều nào → **loại khỏi danh sách auto-fix**. Còn nghi ngờ thì hạ xuống mục "cần xác nhận" cho user, KHÔNG tự sửa.
   - **BÁC BỎ hẳn** (loại khỏi báo cáo) thì phải bằng phép đo **của chính mình** bác được MỆNH ĐỀ. Chạy lại phép đo của agent ra số khác mới chỉ hạ CHỨNG CỨ: dạng hỏng hay gặp nhất của agent chạy nhanh là *kết luận đúng, chứng cứ ẩu* — bác theo chứng cứ là bỏ sót bug thật.
   - Đường đo biến nghi ngờ thành bằng chứng (đầy đủ ở Step 3 của `review the last change`, `gitlab-flow`): thư viện ⇒ chạy hàm thật kèm **ca đối chứng**; thư viện đóng ⇒ **tiền lệ** cùng hình dạng trong chính class đó, trên cùng instance/datasource; giá trị A→B ⇒ **hằng số sẵn có** cho B; "thiếu test" ⇒ viết–chạy–xoá test đó; "không thể xảy ra" ⇒ chỉ ra chỗ **code** chặn, không phải tài liệu; fix "fail fast"/chốt chặn/phụ thuộc thứ tự khởi tạo ⇒ **dựng điều kiện hỏng và đo**; giá trị ghép từ ≥2 nguồn ⇒ ca tái hiện là **tích Descartes** các biên, không nguồn nào lấy mặc định.
3. **Finding bị agent tự HẠ** (xuống `Minor`, hoặc vào mục "dưới ngưỡng / đã cân nhắc bỏ qua") cũng qua verify: kiểm lý do hạ theo `Review output` (anh em trong chính MR, tiền lệ là quyết định hay tích tụ, hệ quả còn lại của dòng code, đo đủ từng nguồn, tầng chặn phía sau có đang bật). Verify chỉ soi chiều nâng thì chiều hạ lọt trọn vẹn mà không phát tín hiệu nào.
4. `Minor` chỉ dedup, không cần verify (không auto-fix nên sai cũng không phá code). Mục `Cần xác nhận` của agent đi **thẳng** sang mục cùng tên ở báo cáo Phase 3 — không có gì để verify, đó chính là lý do nó ở đó.

> Danh sách sau verify **ngắn hơn nhiều** so với tổng 4 agent. Đó là dấu hiệu ĐÚNG, không phải review kém. Báo cáo số bị loại để user thấy tầng verify có chạy — tách số bị bác vì **mệnh đề sai** khỏi số được **sửa lại chứng cứ mà giữ nguyên**.

**Phase 3 — Fix + báo cáo**:

1. Fix trực tiếp trong working tree **chỉ `Blocker` + `Major` đã qua Phase 2.5** (trong diff, hoặc ngoại lệ cơ học ngoài diff). `Minor` gom vào mục riêng để user tự quyết, `Nit` bỏ. Không còn finding nào sau verify → báo "Không có vấn đề chặn". Chụp TRƯỚC, đối chiếu TRƯỚC/SAU và soát phạm vi sau fix (mọi điểm ghi state, test double khi đổi chữ ký gọi) theo `fix all issues` bước 3-4 ở `gitlab-flow`. Fix kiểu "làm lỗi ồn ào" thì câu mô tả trong báo cáo/commit phải là chế độ hỏng ĐÃ ĐO, không phải suy từ `throw`. "Đã fix" chỉ dành cho fix qua phép kiểm "fix ĐÚNG" ở Phase 2.5 (tầng schema: đã chạy trên DB có dữ liệu); còn lại ghi "đã viết code sửa, chưa chứng minh có tác dụng" — vòng sửa này là code mới, MR sẽ soi nó như code gốc.
2. **KHÔNG tự commit/push** — để user review changes rồi tự `commit and push` (sẽ hỏi xác nhận push như thường lệ)
3. Tóm tắt theo đúng format này — `Minor` phải có chỗ đứng, nếu không item 1 nói "gom vào mục riêng" mà không có mục nào. Chạy khối "Tự soát chính báo cáo" ở Step 4 của `review the last change` trước khi đưa ra:
   ```
   Đã fix: <N> Blocker, <M> Major — <danh sách file>   (ngoài diff gốc: <F> file)
   Verify: lens 4/4 (Correctness <a> · Security <b> · Efficiency <c> · Quality <d>) → <T> finding thô → <D> vấn đề sau dedup → bác bỏ <K> (mệnh đề sai) · sửa chứng cứ giữ nguyên <E> → còn <R> để fix
   Test/typecheck: <status> — đầu vào: <số file/test thực chạy>; build fail đã phân loại theo mã lỗi
   Chưa làm: <việc> — <lý do cụ thể: không có tool / bị từ chối quyền / môi trường thiếu>

   ### Minor (không fix — user tự quyết)
   #1 [Minor] path/file.cs:15 — <vấn đề>

   ### Ngoài diff (không tự fix)
   #2 [Major] path/other.cs:40 — <vấn đề>. Có từ trước nhánh này

   ### Cần xác nhận (không tự fix)
   #3 path/file.cs:80 — <vấn đề>. Chưa chứng minh được vì <lý do>. Phép đo sẽ phân xử: <…>
   ```
   Dòng `(ngoài diff gốc: <F> file)` là bắt buộc khi F > 0: working tree user sắp commit đã rộng hơn cái họ nhớ.
4. Gợi ý bước tiếp: nếu có fix → `commit and push` rồi `create a merge request`; nếu không có gì cần sửa → `create a merge request` luôn. **Và nói thẳng rằng review này KHÔNG thay được bấm thử**: 4 lens xét tính nhất quán NỘI TẠI của code, nên loại lỗi "code đúng nhưng giả định về thế giới bên ngoài sai" (URL sinh ra có tồn tại không, endpoint có nhận method đó không, trang đích có dựng được không) nằm ngoài tầm. Liệt kê những thao tác người dùng mà diff chạm tới để user biết cần bấm gì. Danh sách rỗng nghĩa là "không thấy mâu thuẫn nội tại", không phải "nhánh lành".

**Lưu ý**:
- Diff > 2000 dòng → review có thể coarse-grained. Khuyến cáo user lần sau chạy sớm hơn (sau mỗi vài commit) thay vì để dồn cuối.
- Giá trị MẶC ĐỊNH của thư viện là điểm mù có hệ thống của mọi review đọc-code — nó không có dòng nào để đọc (Block 2 bước 9).
- Base branch (`master`/`develop`/`dev` thay `main`) → xem mục **Base branch** ở `gitlab-flow`. Nếu `<base>` đã được chốt lúc tạo branch thì tái dùng; nếu chưa rõ (vd review branch tạo ngoài skill) → hỏi user `main` hay `dev` trước khi `git merge-base`.
- Trigger này chuyên review macro. Để review chỉ thay đổi gần nhất → dùng `review the last change`. Để review MR đã push (vai Reviewer) → dùng `review the MR !<N>`.

### "review the MR !<N>" (vai trò Reviewer)

0. **Chạy ở context KHÁC context đã viết code.** Thiên lệch của người viết không nằm ở chỗ thấy hay không thấy, mà ở chỗ *định giá* cái đã thấy ("vô hại", "đã cân nhắc") — thêm lens không sửa được, vì mọi lens chung một context. "Context khác" được thoả bằng **subagent do chính phiên này spawn** — 🚫 không dừng quy trình để bảo user mở session mới. Prompt của subagent: bảo nó gọi `Skill` `gitlab-review` với trigger `review the MR !<N>` (không chép rubric vào prompt — bản chép tay là bản sao thứ hai của một quy trình đang sống, lệch âm thầm khi skill đổi), và **không truyền kết luận, lý lẽ hay danh sách finding của lượt trước** (kể cả mục "đã cân nhắc bỏ qua"). Bạn CHÍNH LÀ subagent đó, hoặc phiên này chưa từng chạm code của MR ⇒ làm thẳng các bước dưới. Buộc phải tự review code của chính mình ⇒ nói rõ trong output đây là tự-review và liệt kê chỗ đã chủ động hạ mức.
1. Yêu cầu `glab` CLI đã cài: kiểm tra `glab --version`
2. Lấy thông tin MR + comment đã có:
   - `glab mr view <N> --comments` (hiển thị cả note/discussion đã có)
   - Số MR do người gõ tay là dữ liệu có thể sai. `state` ≠ opened ⇒ báo trước khi review. User nói các MR là một CẶP/liên đới ⇒ so `source_branch` (hoặc tham chiếu chéo trong mô tả); lệch ⇒ tìm MR đang mở cùng source branch ở repo kia (`glab api "projects/:id/merge_requests?state=opened&source_branch=<branch>"`): khớp duy nhất ⇒ dùng và nói rõ ở dòng đầu output; mơ hồ ⇒ hỏi.
3. **Ghim revision MỘT lần**, rồi **BẮT BUỘC** lấy diff từ remote bằng `glab mr diff <N>`:
   ```bash
   glab api "projects/:id/merge_requests/<N>" > .git/mr_<N>.json        # sha, source_branch, target_branch
   MR_SHA=$(python -c "import json;print(json.load(open('.git/mr_<N>.json',encoding='utf-8'))['sha'])")
   git fetch origin <source-branch> <target-branch>     # doc stderr: co "error:"/"couldn't set" => origin/<target> la dau CU
   git cat-file -e "$MR_SHA^{commit}" && echo "da co revision $MR_SHA"
   glab mr diff <N> > .git/mr_<N>.diff
   grep -E '^(---|\+\+\+) ' .git/mr_<N>.diff | grep -v /dev/null | sed -E 's#^(---|\+\+\+) ([ab]/)?##' | sort -u > .git/mr_<N>.glab_files
   git diff --no-renames --name-only origin/<target-branch>..."$MR_SHA" | sort -u > .git/mr_<N>.git_files
   diff .git/mr_<N>.glab_files .git/mr_<N>.git_files && echo "danh sach file khop"   # glab mr diff co the bi CAT
   wc -l .git/mr_<N>.diff; grep -c '^@@' .git/mr_<N>.diff                    # phep dem doc lap
   git log --no-merges --format='%h %an <%ae> %ad %s' --date=short origin/<target-branch>.."$MR_SHA"
   ```
   - Danh sách file lệch ⇒ `glab mr diff` đã bị cắt (thiếu im lặng): dùng `git diff origin/<target-branch>..."$MR_SHA"` làm nguồn chính và ghi trong output "`glab mr diff` thiếu <k> file". Fetch báo `error:`/`couldn't set` ⇒ lấy đầu target bằng `git ls-remote origin refs/heads/<target-branch>`, `git fetch origin <sha>` và dùng SHA đó thay `origin/<target-branch>`.
   - Diff gộp nói "cái gì đổi", không nói "vì sao nó ở đây": đọc danh sách commit ở trên, `git show --stat` từng commit. Tác giả lạ (`HOST\user`, email không phải danh tính git thật), ngày ngoài khoảng của ticket, hoặc đụng file không liên quan tiêu đề MR / config máy dev ⇒ commit lạc bị cuốn vào nhánh — finding, kể cả khi mô tả MR không nhắc.
   - Mọi lệnh đọc sau đó dùng **`$MR_SHA`, không bao giờ `FETCH_HEAD`**: `FETCH_HEAD` là MỘT slot dùng chung cho cả repo — bất kỳ fetch nào (kể cả GUI của user) cũng dời nó, và `git show FETCH_HEAD:file` vẫn trả nội dung hợp lệ, chỉ là của revision khác. Hai phép đọc cùng revision mà mâu thuẫn ⇒ nghi REF trước, đừng nghi code. SHA này phải xuất hiện nguyên văn trong output (bước 5).
   - Output của `glab mr diff` **không có header `diff --git`** như `git diff` — `grep '^diff --git'` trả 0 dòng trông y hệt "MR không đổi file nào". Luôn kèm phép đếm độc lập ở trên để phân biệt "không có" với "hỏi sai".
   - **KHÔNG** thay `glab mr diff` bằng `git diff <base>...<source>` so với branch local — `main` (hoặc base) ở local có thể stale, dẫn tới review nhầm hàng trăm commits đã có sẵn trên remote. Cần `git diff` (vd để lấy stat) thì so với `origin/<base-branch>` sau khi fetch, không phải branch local.

   > **3 mức độ sâu — chọn theo nhu cầu:**
   >
   > | Mức | Context có được | Cần checkout? |
   > |---|---|---|
   > | **Inline** | Chỉ text diff từ `glab mr diff <N>` | Không |
   > | **Grounded** ← *mặc định* | Full file + caller tại đúng revision của MR | **Không** |
   > | **Deep** | Grounded + 4-agent ensemble + verify | **Không** — trừ khi dùng `/code-review` built-in |
   >
   > **Grounded — đọc full file mà KHÔNG đụng working tree.** Đây là mức mặc định vì nó gỡ được điểm yếu chí mạng của review chỉ-nhìn-diff mà không bắt user rời nhánh đang làm dở:
   > ```bash
   > git show "$MR_SHA:path/to/file.js"          # full file tai dung revision cua MR
   > git grep -n "functionName" "$MR_SHA"        # caller trong TOAN repo tai revision do
   > git ls-tree -r --name-only "$MR_SHA"        # cay file (phep lay hieu tap file, phan giai file duoc vien dan)
   > ```
   > Git Bash (MSYS): `export MSYS_NO_PATHCONV=1` trước các lệnh `<rev>:<path>` — path bắt đầu bằng `.` (`.changeset/`, `.gitlab-ci.yml`, `.env.example`) bị dịch thành path list Windows; dấu hiệu: "unknown revision" mà chuỗi in ra có `\` và `;` — lỗi của đối số, không phải ref chưa fetch.
   > ⚠️ **TUYỆT ĐỐI KHÔNG đọc file ở working tree để lấy context cho MR.** Working tree đang ở nhánh khác — nội dung file có thể khác hoàn toàn với revision của MR. Ghép diff của MR với context của nhánh khác cho ra finding sai mà nghe rất thuyết phục. Chỉ đọc qua `$MR_SHA:` (hoặc sau khi đã checkout đúng nhánh MR). Kỷ luật này đi theo PHÉP ĐO, không theo repo: grep caller sang repo thứ hai thì áp Block 2 bước 5 (MR đồng hành), không đọc working tree của repo đó.
   > ⚠️ Fetch fail (offline / chưa auth) → tụt về mức **Inline** và **nói rõ với user**, đừng im lặng đọc working tree thay thế.
   >
   > **Deep** — 4-agent ensemble mà KHÔNG checkout: chạy Phase 1 bước 6 → Phase 2.5 của `review the whole branch` với input là `.git/mr_<N>.diff`, và chép `$MR_SHA` (SHA đầy đủ) + ba lệnh đọc ở khối Grounded vào prompt từng agent, kèm cảnh báo tuyệt đối không đọc working tree. Cần chạy build/test tại revision đó ⇒ theo đúng khối **Build/test tại revision MR** ngay dưới — working tree của user vẫn nguyên. Vai Reviewer không auto-fix code của người khác: kết quả Phase 2.5 đi thẳng vào bước 4-5 để post. Checkout (`glab mr checkout <N>`, `git checkout`, GUI) chỉ khi thật sự dùng `/code-review` built-in — công cụ đó đọc working tree.
   >
   > **Build/test tại revision MR — công thức DUY NHẤT được phép** (Grounded khi muốn xác nhận build/test; Deep). Chép NGUYÊN khối này vào prompt của mọi agent được phép build/test hoặc chạy đột biến — brief cấm working tree mà không đưa đường thay thế là mời agent tự chế (#425):
   > 1. Chỉ cần ĐỌC file ở revision khác ⇒ `git show`/`git grep` như khối Grounded. Không dựng worktree.
   > 2. Cần build/test ⇒ `git -c core.longpaths=true worktree add --detach "<thư-mục-ngắn-NGOÀI-repo>/wt-<N>" "$MR_SHA"`. KHÔNG nuốt stderr; kiểm thư mục worktree tồn tại trước khi chạy bất cứ gì — dựng hỏng thì báo "không dựng được", không đọc kết quả test (hỏng im lặng trông giống test đỏ).
   > 3. Cài dependency TẠI CHỖ trong worktree (`npm ci` / `pnpm install --frozen-lockfile` / `dotnet restore`). 🚫 **TUYỆT ĐỐI không symlink/junction `node_modules` hay bất kỳ thư mục dùng chung nào** từ cây làm việc sang worktree: trên Windows xoá đệ quy đi THEO link, và workspace monorepo có symlink trỏ ngược về `apps/*`, `packages/*` — đã xoá 869 file đã track (#425) và ruột 99 gói làm sập dev server của phiên khác (#262).
   > 4. Chạy suite TUẦN TỰ, không song song với typecheck/lint (test nhạy thời gian đo cả tải máy). Test đỏ ⇒ chạy lại riêng nó, cô lập, ≥2 lần, và `git diff --quiet origin/<target-branch>..."$MR_SHA" -- <file-test> <file-nó-phủ>`: chỉ đỏ lặp lại được VÀ chạm diff mới là finding; còn lại báo "flaky/môi trường, ngoài diff".
   > 5. Đột biến/mutation chỉ làm trong worktree, không bao giờ trên working tree của user.
   > 6. Dọn: `git worktree remove "<wt>"`; long path trên Windows ⇒ `Remove-Item -LiteralPath "\\?\<wt>" -Recurse -Force` rồi `git worktree prune`.
   > 7. Agent xong ⇒ phiên chính kiểm `git status` của repo user, `git worktree list`, và nếu có dev server đang chạy thì gọi một route THẬT. Đếm thư mục/đếm file không phải kiểm nội dung (#262). Ghi kết quả build/test — hoặc "không chạy" — vào output review.
   >
   > **Base cho deep review = target branch của MR**, KHÔNG hỏi `main`/`dev`. Lấy bằng `glab mr view <N>` (field targetBranch) — dự án main-chính ra `main`, dự án dev-chính ra `dev`. Dùng đúng nhánh đó làm `<base>` khi `review the whole branch` (đây là ngữ cảnh reviewer: base đã xác định sẵn từ MR, không thuộc trường hợp "hỏi user" ở Phase 1).
4. **Phân nhánh theo HAI trục đo được** — đừng hỏi proxy "MR có comment chưa", vì nó gộp hai câu hỏi rời nhau:
   - **Trục 1 — đã có lượt review ĐỘC LẬP chưa?** Comment do chính author của MR đăng (dán output tự-review) không tính: mục đích của trigger này là con mắt thứ hai.
   - **Trục 2 — code đã đổi kể từ lượt đó chưa?** So `$MR_SHA` với SHA ở dòng `**Reviewed at:**` của comment review cuối. Comment cũ không ghi SHA ⇒ mới tụt xuống timestamp, và ép cả hai vế về UTC trong chính câu lệnh (note `created_at` là UTC `…Z`, còn `git log` mặc định in giờ địa phương kèm offset): `TZ=UTC git log --date=iso-strict-local --format='%H %cd' <range>`, so bằng công cụ, **không so hai chuỗi thời gian khác gốc múi giờ bằng mắt**.

   | Có commit mới? | Đã có review độc lập? | Làm gì |
   |---|---|---|
   | — | Không (chưa có comment, hoặc chỉ có self-review của author) | **(A)** trên head; đã có số issue thì đánh số tiếp nối; ghi ở đầu output: đây là pass Reviewer độc lập, không phải review tiếp nối |
   | Có | Có | **(B)** review tiếp nối |
   | Không | Có | Báo "chưa có commit mới kể từ review tại `<sha>`" và hỏi **MỘT lần** có muốn pass thứ hai không. Trigger được gọi lại với đầu vào không đổi ⇒ không hỏi nữa, chạy **(A)** đầy đủ trên head: "không có commit mới" không có nghĩa là "đã soi hết" |

   **(A) Review mới trên head**:
   - **Nạp context trước đã** (mức Grounded — xem bảng 3 mức ở trên): `git show "$MR_SHA:<file>"` để đọc **full file** của mọi file trong diff, `git grep -n <symbol> "$MR_SHA"` để truy caller. Cũng đọc `CLAUDE.md` tại `$MR_SHA` nếu có. Áp Block 2 bước 3–9 của `review the whole branch` (Grounding ngoài diff: field bị gỡ bên ghi, kết luận phủ định, MR đồng hành ở repo khác, MR dựng từ mẫu, chú thích biện minh/viện dẫn) ở mức này.
   - Review toàn bộ diff theo tiêu chí Step 2 của mục "review the last change" (ở `gitlab-flow`, gồm `Flag kích hoạt theo tình huống`). Có full file → **áp nguyên Step 3** ("chứng minh bằng code đã đọc", đường đo, ca tái hiện) như bình thường
   - Lời khai kiểm chứng của tác giả trong MR ("typecheck sạch", "N test pass", bảng đo) chỉ là bằng chứng khi chứng minh được **đầu vào không rỗng** và phủ đúng file của MR — đo độ phủ (tại worktree `$MR_SHA` nếu cần chạy), đừng tái chạy cùng lệnh rồi đọc exit code: cùng sai theo cùng một cách trông như đã kiểm chứng độc lập.
   - ⚠️ Chỉ khi fetch thất bại và phải tụt về mức **Inline** (chỉ có text diff) thì mới nới Step 3: finding nào cần context ngoài diff ghi **"cần xác nhận"**, đừng khẳng định. Nói rõ trong output là review chạy ở mức Inline
   - Liệt kê issues `#1 [Severity] file:line — <vấn đề>. Đề xuất: <fix>`
   - Verdict map thẳng từ severity: còn `Blocker`/`Major` → `REQUEST_CHANGES` · chỉ còn `Minor` → `COMMENT` · sạch → `APPROVE` (nói rõ "không có vấn đề chặn", không bịa Nit để có cái mà comment)

   **(B) Review tiếp nối** — KHÔNG review lại từ đầu:
   - Đọc kỹ comment cũ, trích xuất danh sách issue đã raise (`#1`, `#2`, ...) kèm verdict gần nhất
   - Xác định mốc của lần review trước, theo thứ tự: SHA ở dòng `**Reviewed at:**` → commit SHA mà reviewer reference → `created_at` của note review cuối (so theo UTC như trục 2)
   - Lấy commit mới từ sau mốc đó: `git log <last-reviewed-sha>.."$MR_SHA"`
   - **Đo phạm vi THẬT trước khi đọc code.** Commit mới ≠ việc mới của tác giả: nhánh sống lâu có commit đồng bộ từ nhánh đích, kể cả commit "resolve conflict" chỉ có 1 parent (nên `--no-merges` không lọc được):
     ```bash
     B=$(git merge-base origin/<target-branch> "$MR_SHA")
     git diff --stat "$B" "$MR_SHA"     # = viec CUA MR, da tru moi thu den tu nhanh dich
     ```
     Chỉ đọc kỹ file nằm trong GIAO của (file trong commit mới) và (file trong diff từ merge-base). Phân loại commit theo `diff --stat` so với merge-base, không theo tiêu đề hay số parent. Diff từ merge-base có file lạ hoặc dòng bị XOÁ mà MR không có lý do xoá ⇒ dấu hiệu lượt resolve conflict đã revert code của nhánh đích — finding.
   - **Đối chiếu issue cũ**: với mỗi issue `#N` đã raise, kiểm tra trong commit/diff mới xem đã được fix chưa — "ca xấu còn tái hiện không?" chưa đủ: áp phép kiểm "fix ĐÚNG" ở Phase 2.5 bước 2 của `review the whole branch` (chỗ fix chạm + consumer phía sau, cổng chặn thử false-positive trên đường hợp lệ, lớp lỗi quét đủ chỗ, schema chạy trên DB có dữ liệu). Đánh dấu:
     - `✓ Resolved #N` — đã fix đúng
     - `❌ Still open #N` — chưa fix, hoặc fix sai/chưa đủ — kèm lý do
     - `⚠️ Partially #N` — fix một phần, kèm điều còn thiếu
     - `🔒 Accepted #N` — giữ nguyên CÓ CHỦ Ý, owner đã chấp nhận rủi ro. Chỉ dùng khi có **con trỏ tới quyết định** (ai chấp nhận, ở đâu: comment/approve/ticket). Chỉ có lời khai của tác giả MR ⇒ ghi `🔒 Accepted (chưa verify — theo comment của tác giả)` và nói rõ reviewer không kiểm chứng được
   - **Issue mới phát sinh từ commit mới**: đánh số tiếp theo (`#N+1`, `#N+2`, ...), không tái sử dụng số cũ. Diff của vòng sửa là **code MỚI**, chưa qua lens nào ⇒ review nó đủ lens theo `Review lenses` (ít nhất Correctness + Quality & Reuse/quy ước), không chỉ "fix có khớp issue cũ không". Commit do chính reviewer/phiên này viết (`%ae`/`%ce` trùng `git config user.email`) ⇒ lens quy ước bắt buộc chạy ở context riêng, đọc file cùng thư mục để học quy ước thật; output ghi rõ issue nào được resolved bởi commit của reviewer
   - **KHÔNG** review lại các phần code không thay đổi từ lần review trước (trừ khi liên quan trực tiếp tới issue cũ) — review tiếp nối thu hẹp phạm vi CODE, không thu hẹp các góc nhìn
   - Verdict: `APPROVE` nếu mọi `Blocker`/`Major` cũ đã `✓ Resolved` hoặc `🔒 Accepted` và không có `Blocker`/`Major` mới; `REQUEST_CHANGES` nếu còn `Blocker`/`Major` ở trạng thái `❌ Still open`/`⚠️ Partially` hoặc mới phát sinh; `COMMENT` cho phần còn lại. ⚠️ **`Minor` chưa fix và `🔒 Accepted` KHÔNG chặn `APPROVE`** — tính vào thì MR treo vĩnh viễn (severity đo mức nghiêm trọng, không đo trạng thái công việc). Nhưng issue `Accepted` phải được liệt kê lại ở mọi lượt sau: chấp nhận rủi ro không có nghĩa là ngừng nhắc
5. Output format thống nhất — dòng `**Reviewed at:**` là **bắt buộc** ở mọi lượt (A lẫn B): output này là ĐẦU VÀO của lượt sau, nên phải tự mang mốc mà lượt sau cần, không trông vào việc phiên sau còn nhớ. Trước khi đưa ra, chạy khối "Tự soát chính báo cáo" ở Step 4 của `review the last change` (con số đo được vs câu chữ mô tả cùng hiện tượng):
   ```
   ## Review !<N> (lần thứ <K>)

   **Verdict:** APPROVE | REQUEST_CHANGES | COMMENT
   **Reviewed at:** <full-sha của $MR_SHA>

   ### Trạng thái issue cũ        ← chỉ có ở mode (B)
   - ✓ Resolved #1
   - ❌ Still open #2 — <lý do>
   - ⚠️ Partially #3 — <còn thiếu>
   - 🔒 Accepted #4 — <ai chấp nhận, ở đâu> (hoặc "chưa verify — theo comment của tác giả")

   ### Issue mới
   - #N+1 [Blocker] `path/to/file.js:42` — <vấn đề>. Đề xuất: <fix>
   - #N+2 [Major]   `path/to/file.js:88` — <vấn đề>. Đề xuất: <fix>

   ### Minor (không chặn merge)
   - #N+3 [Minor]   `path/to/file.js:15` — <vấn đề>

   ### Cần xác nhận
   - #N+4 `path/to/file.js:60` — <nghi ngờ>. Phép đo sẽ phân xử: <…>
   ```

### "post review result to the MR"

> 🚫 **KHÔNG chèn AI attribution** vào comment (Co-Authored-By Claude, 🤖 Generated with, link claude.com, ...). Comment = chỉ nội dung review thuần, KHÔNG signature/footer.

1. Lấy chính output Markdown từ bước "review the MR" trước đó (đã đúng format, không cần soạn lại). Nếu là review tiếp nối (mode B), giữ nguyên cả phần "Trạng thái issue cũ" — đó là context quan trọng cho dev. **Giữ nguyên dòng `**Reviewed at:** <sha>`** — lượt review sau lấy mốc từ đó.
2. **Self-check trước khi đăng — phải CHẶN được lệnh post**, không chỉ in ra: chạy ở tool call RIÊNG, chỉ sang bước 3 khi dòng cuối in ra là `PASS` — crash hay không in gì cũng là CHẶN. 🚫 Không bao giờ nối check với post bằng `;`. Tên file repo (`CLAUDE.md`, `.claude/`) được bóc trước nên không khớp nhầm:
   ```bash
   python -c "import re,sys;t=re.sub(r'\.?claude(\.md|/)','',open(sys.argv[1],encoding='utf-8').read(),flags=re.I);m=[l for l in t.splitlines() if re.search(r'claude|anthropic|\U0001F916|generated with|co-authored-by',l,re.I)];print(*m,sep='\n') if m else None;print('BLOCKED: attribution' if m else 'PASS');sys.exit(1 if m else 0)" review.md
   ```
   Viết bằng MỘT lệnh `python -c`, KHÔNG dùng `{ ...; }` hay `if ...; then ... fi`: ở chế độ tự động
   (`claude -p` + `--allowedTools`) bộ kiểm quyền của Claude Code từ chối cả hai dạng đó dù đủ quyền
   cho từng lệnh con (đã đo) — cổng bị bỏ qua đúng nơi không có người nhìn. Đường dẫn file truyền dạng
   `/` hoặc trong nháy đơn: bash nuốt `\` của đường dẫn Windows và python crash (= chặn, nhưng vô ích).
   Có → xóa rồi chạy lại. Quét bí mật theo **giá trị thật** (mục `Safety rules` ở `gitlab-flow`) — review hay trích nguyên dòng config/code, và comment là nội dung đã publish.
3. Đăng comment. Output review là markdown dài tiếng Việt ⇒ **KHÔNG đẩy qua argv** (argv/pipe của PowerShell làm hỏng dấu; hỏng theo shell chứ không theo lệnh, và comment đã đăng thì khó rollback êm). Đi đường file — lý do đầy đủ: `create a merge request` bước 3 ở `gitlab-flow`:
   ```bash
   # review.md ghi bang tool Write (UTF-8, KHONG BOM); python chi nhan path TUONG DOI (khong dung /c/... cua MSYS)
   python -c "import json;json.dump({'body':open('review.md',encoding='utf-8').read()},open('note.json','w'))"
   glab api --method POST -H "Content-Type: application/json" "projects/:id/merge_requests/<N>/notes" --input note.json
   glab api "projects/:id/merge_requests/<N>/notes?sort=desc&per_page=5"      # doc lai truoc khi coi la xong
   ```
   - Thiếu `-H "Content-Type: application/json"` ⇒ HTTP 415 và **không post gì**. Lệnh báo lỗi hay không, **đọc lại danh sách note**, so độ dài + đoạn đầu/cuối với `review.md`, rồi mới coi là xong hoặc mới thử lại — retry mù là comment trùng.
   - Text ngắn không dấu: `glab mr note <N> --message "<text>"` vẫn là lệnh chính. Trên glab 1.93.0 cờ `--message` in cảnh báo deprecated (exit 0, cùng luồng với URL kết quả), còn `glab mr note create` mà cảnh báo chỉ tới tự khai là **experimental** — chưa đổi, giữ lệnh cũ cho tới khi bản mới rời nhãn experimental hoặc bản cũ bị gỡ.
4. Nếu APPROVE: **cổng danh tính trước** `glab mr approve <N>` (chạy lúc bước 2, trước khi post, để note mang được kết quả) — review độc lập về context không phải độc lập về danh tính, và GitLab chỉ thấy danh tính:
   ```bash
   glab api user                                                   # username, email cua tai khoan glab
   B=$(git merge-base origin/<target-branch> "$MR_SHA")
   git log --format='%h %ae %ce %s' "$B".."$MR_SHA"                # phan code dang duoc duyet
   ```
   `author.username` của MR (`.git/mr_<N>.json`) trùng username đó, HOẶC có commit trong phạm vi mà `%ae`/`%ce` trùng email đó hay `git config user.email` (reviewer đã tự push fix) ⇒ **KHÔNG approve**: note ghi "approve để reviewer khác — <tác giả là tài khoản này / các fix #… do reviewer tự push>", báo user; chỉ approve khi user ra lệnh rõ sau khi thấy lý do.
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
  - Các lệnh `glab` ở đây đã kiểm trên **glab 1.93.0** (ghi lại `glab --version` khi thấy lệnh lệch hành vi). Cảnh báo `deprecated` / `use X instead` trong output là tín hiệu bảo trì skill, không phải nhiễu để `tail` bỏ — kể cả khi lệnh thành công; kiểm `--help` của lệnh được chỉ tới trước khi đổi.
