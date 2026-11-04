---
name: gitlab-flow
description: Use when the user references a Jira task ID (WRA-40, HNCW-311, ...) or types "start a task", "create branch from task", "rename branch", "review the last change" / "review change" / "review change simplify", "commit and push" (± --quick), "create a merge request", "fix all issues" / "fix issue #N", "merge main vào nhánh feature" / "gỡ conflict" / "merge main bị conflict" (gỡ conflict khi merge main vào nhánh feature đang mở MR). NOT for the reviewer-side triggers "review the whole branch", "review the MR !N", "post review result to the MR", "merge the request" — those need the add-on skill gitlab-review.
---

# GitLab Flow (Jira → Code → MR → Merge)

Quy trình chuẩn cho một feature/bugfix mới. Có 2 vai trò: **Developer** (người làm task) và **Reviewer** (người review MR). Skill này hướng dẫn Claude thực hiện đúng từng bước theo prompt mà user gọi.

> 📦 **Phần vai Reviewer nằm ở skill riêng.** 4 trigger `review the whole branch`, `review the MR !<N>`, `post review result to the MR`, `merge the request` đã chuyển sang skill **`gitlab-review`** (add-on, chỉ Lead cài). Skill này giữ nguyên `Conventions` mà `gitlab-review` tham chiếu tới. Không cài `gitlab-review` ⇒ **không có** 4 trigger đó — nhờ Lead chạy hộ, đừng tự dựng lại quy trình.

## Conventions

### Branch naming
- **Default: mọi branch dùng `feature/`** — bất kể task là feature, bug fix, hay hotfix.
- Format: `feature/<TASK-ID>-<short-desc>`
- Bug-fix branches dưới `feature/`: mô tả **trạng thái bug** bằng direction marker (`Duplicate`, `Stale`, `Missing`, `Broken`, `Wrong`, `Slow`) thay vì verb "Fix" — vd `feature/HNCW-311-Duplicate-survey-log`
- **Override**: user chủ động gõ `bugfix/...` hoặc `hotfix/...` trong prompt (Mode A) → skill respect và tạo đúng prefix đó. Skill **KHÔNG** tự động chọn `bugfix/` hay `hotfix/` dựa trên nội dung task.

**Quy tắc `<short-desc>`** (đủ để hiểu task ở first glance, chi tiết để Jira giữ):

| Rule | Detail |
|---|---|
| Độ dài tổng | ≤ 50 ký tự cả branch (target ≤ 40). Vượt → rút thêm |
| Số từ | 2-4 từ key. Filler bị drop |
| Ngôn ngữ | Tiếng Việt không dấu, kebab-case (`-` ngăn cách) |
| Capitalization | **Sentence case STRICT**: chỉ chữ cái đầu của từ đầu tiên trong description viết hoa. **Mọi từ sau (KỂ CẢ viết tắt như `NVKD`, `VAT`, `API`, `JWT`)** đều lowercase. Vd `Bao-cao-ngay-nvkd` (KHÔNG `Bao-cao-ngay-NVKD`), `Vat-discount` (KHÔNG `VAT-discount`), `Gioi-han-domain-account`. TASK-ID giữ nguyên uppercase per Jira convention |
| Drop **type filler** | "Cai-tien", "Update", "Improve", "Fix", "Sua", "Sua-loi", "Them", "Tao", "Add", "Create", "Bo-sung" — đều bỏ, **kể cả trong candidate do chính skill sinh ra** (self-check ở Step 3). Với bug fix, mô tả **trạng thái bug** (`Duplicate`, `Stale`, `Missing`, `Broken`) thay vì verb "Fix". Ticket ô-dù (nhiều tính năng) → đặt theo MẢNG, không giữ lại `Cai-tien` để lấp (Step 2) |
| **KEEP direction marker** | "Cho-phep"/"Allow", "Khong-cho-phep"/"Disallow", "Validate", "Block", "Restrict", "Enforce" — chúng nói **WHAT** behavior. Không có chúng → ambiguous (allow? disallow? validate?) |
| **KEEP context marker** | "Show"/"Display"/"Hide" (UI layer), "Filter"/"Sort"/"Search"/"Calculate" (logic layer), "Sync"/"Migrate"/"Schedule"/"Export"/"Import" (system layer) — chúng nói **TẦNG/CÁCH THỨC** của feature, mà `feature/` prefix không cover. Vd `Show-order-info` rõ hơn `Order-info` (display? backend? API?) |
| Drop scope marker | Tag dạng `[Supermarket - AU]` ở đầu task title KHÔNG đưa vào branch (giữ cho commit scope) |
| Drop constraint phụ | Implementation detail như "áp dụng cho sp non-weight" — bỏ. Đó thuộc commit body / Jira description |
| Ưu tiên giữ | **Direction/Context + Action/Object + Phạm vi** (vd `Allow-qty-0-checkin-checkout`, `Show-order-info-uber-doordash`). Mục tiêu: đọc 1 phát hiểu ngay, không cần Jira |

> ⚠️ **Quy ước trong bảng này là GIẢ THUYẾT về repo, không phải sự thật về repo.** Tên nhánh sai
> quy ước vẫn tạo, push, mở MR được — nó chỉ lộ ra khi có người đối chiếu lịch sử. Nên trước khi
> đề xuất tên (Step 2 của `create branch`), đo quy ước thật:
> ```bash
> git branch -r | sed 's|^ *origin/||' | grep -oE '^[A-Za-z]+/' | sort | uniq -c | sort -rn   # tổng theo lịch sử
> git log --merges -20 --format=%s origin/<base>                                           # quy ước HIỆN HÀNH
> ```
> Tiền tố đa số khác `feature/` → nêu con số ngay trong câu hỏi chọn tên, **không tự đổi** (nhánh
> cũ có thể là di sản). Tổng số và merge gần nhất mâu thuẫn → merge gần nhất là quy ước hiện hành.

**Ví dụ áp dụng**:

| Task title (Jira) | ✓ Good branch | ✗ Quá dài / sai |
|---|---|---|
| `WRA-40 Giới hạn domain account khi login` | `feature/WRA-40-Gioi-han-domain` | `feature/WRA-40-Gioi-han-domain-account-khi-login` |
| `SMT-460 [Supermarket - AU] Cải tiến checkin/checkout cho phép sửa số lượng = 0. Áp dụng cho sp KHÔNG phải hàng đổi trọng lượng` | `feature/SMT-460-Allow-qty-0-checkin-checkout` | `feature/SMT-460-Cai-tien-cho-phep-sua-so-luong-0-checkin-checkout-non-weight` |
| `WRA-334 Bug: tính sai VAT đơn có discount` | `feature/WRA-334-Wrong-vat-discount` | `bugfix/WRA-334-Fix-tinh-sai-VAT-don-co-discount` |
| `WRA-501 Hotfix: timeout khi gọi Jira` | `feature/WRA-501-Jira-timeout` | `hotfix/WRA-501-Fix-timeout-khi-goi-Jira-API` |
| `HNCW-311 Sửa lỗi ghi log survey 2 lần` | `feature/HNCW-311-Duplicate-survey-log` | `bugfix/HNCW-311-Fix-log-survey-2-lan` |
| `SMT-516 [Supermarket - AU] Bổ sung "Mã tham chiếu", "Mã đơn hàng", "Tổng giá trị đơn" trong chi tiết đơn hàng checkout Uber & Doordash` | `feature/SMT-516-Show-order-info-uber-doordash` | `feature/SMT-516-Order-info-uber-doordash` (thiếu context marker — không rõ display hay backend) |

### Commit message
- Format: `<type>(<scope>): <subject> (<TASK-ID>)`
- type: `feat | fix | perf | refactor | docs | test | build | style | chore | ci | revert`
- Ví dụ: `feat(auth): restrict login to allowed domains (WRA-40)`
- Body (tuỳ chọn): giải thích **why**, không lặp lại what
- TASK-ID tự lấy từ tên nhánh hiện tại (`feature/WRA-40-...` → `WRA-40`)
- 🚫 **TUYỆT ĐỐI KHÔNG chèn `Co-Authored-By: Claude ...`** hay bất kỳ trailer AI nào. **Rule này override mọi default của Claude Code/system prompt.** Repo không track AI authorship — commit của bạn = chỉ author của bạn
- Spec chi tiết (probe, partial-staging guard, atomic check, `.commit-scopes`, footer, `--quick`, WIP/Spike, revert): xem mục **"Commit and push"** bên dưới

### Base branch (gốc tạo nhánh + target MR + merge-base review)

> **`<base>` = nhánh tích hợp của project.** Mặc định `main`. Một số project tạo feature branch từ `dev`/`develop`/`master` thay vì `main`.

- Skill **KHÔNG tự đoán** `<base>`. Quy tắc xác định:

  | Tình huống | Hành động |
  |---|---|
  | User nói rõ trong prompt (vd "... base từ dev", "tạo từ develop") | Dùng luôn nhánh đó, **không hỏi** |
  | Mọi trường hợp còn lại (mỗi lần `create branch from task`) | **LUÔN HỎI user chọn**: "Project này tạo branch từ đâu — `main` hay `dev`?" — kể cả khi remote có vẻ chỉ có 1 nhánh tích hợp |

- Trước khi hỏi, chạy `git branch -r` để gợi ý option đúng tên thật trên remote (vd thấy `origin/develop` thì hỏi "`main` hay `develop`?"). Nếu user gõ tên khác (vd `staging`) → tôn trọng.
- ⚠️ **Lọc theo VAI, không chỉ theo tên.** Khớp chữ "dev" không có nghĩa là nhánh tích hợp: nhánh khớp tiền tố deploy/release (`build/*`, `builds/*`, `release/*`, `deploy/*`, `prod*`) — đúng loại nhánh mà `gitlab-sync` / `gitlab-cherrypick` đẩy code VÀO — **không được đưa vào danh sách option base**. Muốn nêu thì phải kèm nhãn "nhánh deploy, không phải nhánh tích hợp". Rẽ nhánh feature từ đó là mang commit đặc thù deploy vào MR ngược về `main`. Cùng bộ lọc áp cho mọi chỗ hỏi lại `<base>` (target MR, `gitlab-review`).
- ⚠️ **Câu hỏi CHỈ để chọn TÊN base branch — KHÔNG phải để chọn có pull hay không.** Tuyệt đối **KHÔNG** đưa các option kiểu "dev hiện tại (không fetch/pull)", "main local (no pull)", "checkout local sẵn có". Mỗi option = một tên branch (`main`, `dev`, `develop`...). Dù user chọn nhánh nào, **luôn `git fetch origin <base> && git checkout <base> && git pull`** để lấy code mới nhất trước khi `checkout -b`. Tạo branch từ base chưa pull = sai (branch ra từ commit cũ).
  - Ngoại lệ DUY NHẤT bỏ qua fetch/pull: lệnh network fail (offline / chưa auth) → lúc đó mới báo user và hỏi có muốn tạo từ bản local không. Không bao giờ đưa "no pull" thành lựa chọn mặc định.
- Một khi user đã chọn `<base>` cho lần tạo branch này, dùng **nhất quán** cho cả vòng đời branch đó: checkout gốc khi tạo, `--target-branch` của MR, `git merge-base <base> HEAD` khi review whole branch (skill `gitlab-review`), và checkout sau khi merge.
- ⚠️ **Bộ nhớ `<base>` chỉ tồn tại trong 1 session.** Nếu trigger sau (vd `create a merge request`, hoặc `review the whole branch` / `merge the request` ở skill `gitlab-review`) chạy ở **session khác** với lúc tạo branch → skill **KHÔNG còn nhớ** `<base>`. Lúc này **phải HỎI lại** user (`main` hay `dev`), tuyệt đối **không mặc định `main`**. Chỉ bỏ qua hỏi nếu `<base>` đã được xác lập trong chính session đang chạy.

### Target branch
- MR luôn merge vào `<base>` (mặc định `main`) trừ khi user chỉ định khác

### Output language (review & report)

**Mặc định tiếng Việt** cho mọi output của các trigger review/report — kể cả khi user gõ trigger bằng tiếng Anh ("review the last change", "review change simplify"...). User KHÔNG cần phải nhắc lại bằng tiếng Việt mới nhận được output tiếng Việt.

| Áp dụng cho | Phần phải tiếng Việt |
|---|---|
| `review the last change` / `review change` (± simplify) | Tóm tắt Step 0 simplify pass + danh sách issue `#1`, `#2`... (vấn đề + đề xuất fix) |
| Tóm tắt sau `fix all issues` | Danh sách issue đã fix + đề xuất commit message |

> Mục này cũng là nguồn `Output language` cho skill **`gitlab-review`** — 4 trigger vai Reviewer ở đó (`review the whole branch`, `review the MR !<N>`, `post review result to the MR`, `merge the request`) áp **cùng** quy tắc trên. Chúng không có dòng riêng trong bảng này: `gitlab-review` gói cả 4 vào **một câu duy nhất** ở đầu skill đó thay vì liệt kê từng trigger.

**Ngoại lệ giữ tiếng Anh** (không Việt hóa):
- `type`/`scope` trong commit message (chuẩn CC: `feat`, `fix`, `auth`, `billing`...)
- Tên technical/identifier: tên file, function, biến, branch, MR title prefix
- Status keyword cố định: `APPROVE` / `REQUEST_CHANGES` / `COMMENT`, `✓ Resolved` / `❌ Still open` / `⚠️ Partially` / `🔒 Accepted`
- Tên agent / role / tool: `Reuse`, `Quality`, `Efficiency`, `glab`, `git`

**Switch language**: user trả lời / tiếp tục bằng ngôn ngữ khác (English chẳng hạn) → từ message đó trở đi mới đổi sang ngôn ngữ user dùng. Không tự đoán "trigger English ⇒ output English".

### Review output (áp dụng cho MỌI trigger review)

Áp dụng cho `review the last change` (skill này) và cho `review the whole branch` / `review the MR !<N>` (skill **`gitlab-review`**, chỉ Lead cài). Đây là **nguồn duy nhất** của bảng severity. ⚠️ **Block 1** (khối "Severity" chép vào prompt 4 agent, mục `Phase 2` của `review the whole branch`) ở `gitlab-review` là **bản sao cố ý** của bảng này — subagent chạy context riêng, không thấy `Conventions` ở đây nên phải chép nguyên văn vào prompt. Sửa bảng severity ở đây thì phải sửa luôn Block 1 ở `gitlab-review`.

| Severity | Gồm | Hành động |
|---|---|---|
| **Blocker** | Sai logic vs task, lỗ hổng security, mất data, crash | Fix |
| **Major** | Edge case có khả năng xảy ra thật, N+1 / perf hot path, race condition | Fix |
| **Minor** | Naming, code thừa, abstraction chưa gọn, comment thừa | **Chỉ liệt kê, KHÔNG fix** |
| **Nit** | Style, format, ý kiến cá nhân | **Bỏ, không báo** |

- Gán severity cho **mọi** finding. Không gán nổi ⇒ chưa đủ rõ ⇒ bỏ.
- **Đổi behavior của hàm/API dùng chung** (signature giữ nguyên nên compiler không bắt được) mà có ≥1 caller **ngoài diff** bị ảnh hưởng ⇒ **`Blocker`**, kể cả khi chưa chứng minh được người dùng cuối thấy lỗi. Lý do: loại này không có tầng nào chặn — không compiler, không test, không reviewer đọc diff.
- Auto-fix (`fix all issues` ở skill này, Phase 3 của `review the whole branch` ở skill `gitlab-review`) **chỉ đụng Blocker + Major**. User gõ đích danh `fix issue #N` thì fix bất kể severity.
- **Không có Blocker/Major → nói "Không có vấn đề chặn" rồi DỪNG.** 🚫 KHÔNG bịa thêm, KHÔNG nâng Nit lên Major để lấp danh sách. **Danh sách rỗng là kết quả hợp lệ.**
- Mỗi finding phải có `file:line` + chứng minh từ code **đã đọc thật**. Không chắc → bỏ, hoặc ghi rõ "cần xác nhận". Bằng chứng gián tiếp thì giữ ở "cần xác nhận" — nhưng trước khi hạ xuống đó, thử các **đường đo** ở Step 3 của `review the last change` (chạy thật, ca đối chứng, tiền lệ, hằng số sẵn có): đo được thì nó thành finding có bằng chứng.
- **HẠ severity cũng là một khẳng định phải chứng minh** — chiều hạ không phát tín hiệu nào, một Blocker bị hạ thành Minor thì không tầng nào bắt lại:
  - **"Hiện trạng repo vốn thế"** chỉ hạ được khi KHÔNG chỗ nào trong chính MR làm đúng luật đó. Grep luật đó trong diff trước: có anh em trong MR làm đúng ⇒ đây là **bỏ sót**, giữ severity theo hậu quả, trích chỗ làm đúng (`file:line`) làm bằng chứng và làm sẵn bản vá.
  - **"Repo đã có tiền lệ"** chỉ bác được finding khi tiền lệ là một QUYẾT ĐỊNH (chỉ ra được ai chọn, lý do còn đúng hôm nay), không phải một TÍCH TỤ (không ai chọn, hoặc nguồn chính là lần trước của mình). Finding bị nêu lại ở vòng review thứ hai thì không bác bằng lại đúng lý do cũ — đưa bằng chứng mới, hoặc sửa.
  - **Severity gán cho DÒNG CODE, không cho lập luận đang dùng nó.** Vai của finding bị bác (bên khác đã vá, đường đi không reachable) ⇒ trước khi hạ, liệt kê mọi consumer của giá trị đó (hiển thị và lưu trữ là hai consumer khác nhau). Còn hệ quả độc lập ⇒ gán theo hệ quả nặng nhất còn lại và viết lại finding theo hệ quả đó.
  - Hạ bằng phép đo ⇒ đo TỪNG nguồn dữ liệu mà finding phủ, và kiểm tầng chặn phía sau có đang BẬT (luật đầy đủ ở Step 3).
- **Severity đo mức nghiêm trọng, KHÔNG đo trạng thái công việc.** Issue mà owner đã chủ động chấp nhận rủi ro mang nhãn `🔒 Accepted #N` (định nghĩa ở mode B của `review the MR !<N>`, skill `gitlab-review`): không chặn verdict, không vào auto-fix, nhưng vẫn được liệt kê lại ở mọi lượt review sau.

> Ngoại lệ: Step 0 của `review change simplify` — user gõ `simplify` = chủ động yêu cầu dọn Minor, nên bước đó được auto-fix Minor.

### Review lenses

Danh mục lens dùng chung. Hai nơi tiêu thụ bảng này:

- `review change simplify` (skill này) — dùng **2 lens cuối** (`Efficiency`, `Quality & Reuse`). Đây là pass dọn dẹp, không phải pass tìm bug.
- `review the whole branch` (skill **`gitlab-review`**, chỉ Lead cài) — dùng **cả 4**, mỗi lens một agent.

⚠️ Đây là **nguồn duy nhất** của định nghĩa lens. `gitlab-review` chép từng dòng — **kèm nhóm "Flag kích hoạt theo tình huống" của đúng lens đó** (ngay dưới bảng) — vào prompt subagent chứ không giữ bản sao. Sửa ở đây là sửa cho cả hai.

| Lens | Tập trung | Severity chủ đạo | Flag điển hình |
|---|---|---|---|
| **Correctness / Task-fit** | Code có làm đúng task không | Blocker, Major | Lệch yêu cầu task, thiếu case so với spec, làm dư ngoài scope, edge case (null/empty/list rỗng/boundary/số âm/unicode), off-by-one, `catch` nuốt lỗi, state nửa vời khi throw giữa chừng, migration không idempotent, timezone/rounding |
| **Security** | Lỗ hổng | Blocker | Thiếu input validation, authz/authn bypass (chặn ở UI mà không chặn ở API), SQL/command injection, path traversal, mass-assignment, secret/token/PII lộ ra log hay response, SSRF, CORS/cookie/session sai |
| **Efficiency** | Performance / resource | Major | N+1, missed concurrency (independent ops chạy tuần tự), hot-path bloat, no-op updates trong polling loops, unnecessary existence checks (TOCTOU), unbounded memory, listener leak, overly broad reads |
| **Quality & Reuse** | Code sạch / tái dùng | Minor | New function duplicates existing helper, inline logic could use existing util (string manipulation, path handling, env checks, type guards), redundant state, parameter sprawl, copy-paste với biến thể nhỏ, leaky abstraction, stringly-typed (raw strings nơi đã có enum/constant), unnecessary JSX nesting, nested conditionals 3+ levels, comment giải thích WHAT |

#### Flag kích hoạt theo tình huống

Mỗi flag chỉ chạy khi diff khớp tình huống ở đầu dòng. Chúng là loại lỗi mà compiler, test và
đọc-diff đều không bắt — nên phải có tên ở đây, không để tuỳ người review nhớ ra.

**Correctness / Task-fit**
- **MR sửa lỗi có nhánh dự phòng** (`catch`, `?? <mặc định>`, `if (!x) return <dự phòng>` do chính bản vá thêm) ⇒ giá trị dự phòng có tái tạo đúng hành vi MR đang sửa không? Đối chiếu lời khai trong chú thích với hàm TIÊU THỤ giá trị đó — đọc riêng từng đầu thì cả hai đều hợp lý. Bảng đo của tác giả gần như chỉ phủ nhánh thành công.
- **Chú thích khai HỢP ĐỒNG** (`không ném`, `trả rỗng khi lỗi`, `idempotent`, `không đụng DB`) ⇒ liệt kê từng lời gọi/`await` trong thân hàm và đối chiếu từng cái. Hàm chép từ hàm sinh đôi ⇒ diff hai THÂN hàm, không diff hai chú thích (chú thích chép nguyên, lời gọi thêm mới thì nằm ngoài lớp bảo vệ).
- **Một khái niệm, hai định nghĩa**: diff thêm danh sách/hằng số trả lời câu hỏi phân loại ("ai là X", "trạng thái nào tính là Y") ⇒ grep theo KHÁI NIỆM (không theo định danh mới) tìm nguồn khác đã trả lời cùng câu hỏi, nhất là cấu hình đọc lúc chạy. Hai nguồn ⇒ `Blocker` (bản hardcode vô hiệu hoá cơ chế cấu hình).
- **Mâu thuẫn nội bộ một file**: liệt kê giá trị sentinel (`0`, `""`, `[]`, `-1`) trong file đã đổi, kiểm cùng loại giá trị có được xử lý nhất quán ở mọi chỗ không; nghĩa của mã thì tra bảng mã phía bên kia, đừng suy từ tên field. Test viết cùng lúc với code có thể ghim luôn cái sai — mỗi `expect` phải có nguồn NGOÀI code đang test.
- **Nhánh đọc phân theo KỲ** (`year == Now.Year`, kỳ hiện tại đọc live / kỳ cũ đọc DB, `Draft`/`Final`) ⇒ truy đường GHI của nhánh kỳ hiện tại: ai ghi, lúc nào. Điều kiện lọc của hàm ghi và điều kiện rẽ của hàm đọc phải PHỦ KÍN nhau; hở một khoảng ⇒ `Blocker` mất dữ liệu có hẹn giờ.
- **Cache/verdict invalidate bằng event handler** ⇒ đúng/sai phụ thuộc thứ tự subscribe, không nằm ở dòng nào trong diff. Invalidate phải xảy ra đồng bộ TẠI CHỖ ĐỌC (so ngữ cảnh hiện tại với ngữ cảnh đã dùng để tính).
- **Mã trạng thái đọc từ hệ ngoài** ⇒ giá trị MẶC ĐỊNH của kiểu rơi vào nhánh nào? `int` với `0 = OK`, `bool HasError`, enum có thành viên đầu là trạng thái tốt ⇒ fail-open ⇒ flag. Câu hỏi nhanh: *trường này biến mất khỏi response thì code tin điều gì?*
- **Dòng tự nhận là chốt chặn** (chú thích "để…", "tránh…", "bảo vệ…", "không lệch…"; bước xác minh trong runbook) ⇒ nó có thể đang MỞ mà chú thích nói đang chặn. Dựng đúng ca nó nói chặn rồi đo (Step 3); không dựng được ca đó cũng là finding.
- **Thêm đường gọi / chế độ chạy / entry point mới** ⇒ grep tài liệu, chú thích, header file cấu hình mô tả đường CŨ như đường duy nhất — kể cả file ngoài diff. Dấu hiệu: "(tự …)", "chỉ cần …", "luôn …", hoặc tên đúng một file trong khi nay có hai.
- **Job / tác vụ định kỳ** (thêm/sửa class job) ⇒ bắt buộc mở file LỊCH CHẠY tương ứng — tìm theo đường dẫn cấu hình job + grep tên class, không suy theo tên file, và nó có thể nằm ở project khác. So độ phân giải của bộ lọc "đến hạn" với chu kỳ chạy: lọc theo ngày + chạy theo phút ⇒ mỗi lượt chạy là một lần lặp. Phải có MỘT trong hai: state "đã xử lý" ghi bền, hoặc cổng khớp giờ. Tác dụng phụ ra ngoài (email/SMS/webhook/thanh toán) thiếu cả hai ⇒ `Blocker`/`Major`. Job anh em trong cùng file lịch là tiền lệ chống lặp của repo. (Báo cáo "gửi lặp": phân biệt *N lượt chạy × 1* với *1 lượt × N phần tử* bằng TỔNG số dòng trên cả khoảng, không bằng cụm timestamp.)

**Security**
- **Cổng kiểm mới nối tiếp cổng cũ, cùng một khoá** (username, mã khách hàng, tenant) nhưng đi đường tra cứu khác ⇒ mở CẢ HAI đường xuống tới tận chỗ chạm DB/API, đối chiếu phép chuẩn hoá khoá (strip prefix, trim, lower, split) — nó thường nằm ở tầng repository, không ở hàm đang đọc. Lệch ⇒ `Blocker`: fail-closed nên hỏng im lặng dưới dạng "từ chối đúng quy trình".
- **Validator nhận nhiều dạng đầu vào** ("cho tiện", "cũng chấp nhận", `||` nới lỏng) mà không chuẩn hoá NGAY tại chỗ kiểm ⇒ đi theo giá trị tới từng chỗ dùng; có chỗ phân biệt được hai dạng (vd chuỗi `'false'` là truthy) ⇒ `Blocker`.
- **Trường trở thành có nghĩa ĐIỀU KHIỂN** (chọn kho/bucket/tenant/endpoint/bảng/hàng đợi — dấu hiệu: bản sửa biến một hằng số thành biến) ⇒ liệt kê MỌI đường giá trị đó vào hệ thống (request body, cột DB, cấu hình, message queue, mặc định) và chỗ kiểm của từng đường. Khoá đường ghi mà để hở đường đọc là chưa xong.

**Efficiency**
- **Diff đặt giới hạn cho lời gọi ra ngoài** (timeout, retry, số lần thử, batch, mức song song) ⇒ tra tầng khởi tạo dùng chung trước (handler/policy mặc định, interceptor): núm mới nằm TRONG hay NGOÀI cái đã có? Tầng dùng chung có retry ⇒ thao tác có idempotent không — lệnh không idempotent bị retry là nhân đôi tác dụng phụ.

**Quality & Reuse**
- **Component/hàm dùng chung mang nội dung của caller ĐẦU TIÊN** (nhãn, danh từ thực thể, đơn vị, thông điệp hardcode trong thân) ⇒ consumer thứ hai sai nhãn mà kiểu/lint đều xanh. Phép kiểm rẻ khi MR bắt đầu dùng lại một component: grep chuỗi ký tự trong thân nó. Tham số hoá bằng tham số **bắt buộc**, không default — default giữ đúng caller cũ và để caller mới sai im lặng.

### Resolve conflict (luật dùng chung)

⚠️ Đây là **nguồn duy nhất** của các luật dưới đây. `gitlab-sync` và `gitlab-cherrypick`
giữ bản tóm tắt ngắn tại chỗ (vì là luật an toàn, phải thấy ngay trong lúc làm) và trỏ về
đây cho phần lý do — sửa ở đây là sửa cho cả ba.

#### Chọn chiến lược merge — KHÔNG lấy-trọn-một-cây để né conflict

`-s ours` / `-s theirs` / `-X ours|theirs` / `checkout --ours|--theirs` cả loạt chỉ dùng khi
**CỐ Ý vứt bỏ toàn bộ nội dung một phía**. Mặc định là merge 3 chiều thường, kể cả khi biết
trước có conflict.

1. **Trước khi chọn cách lấy-trọn-cây, liệt kê state RIÊNG của nhánh đích** (version trong
   `csproj`/`package.json`, changelog, config build/env) và nói rõ với user là chúng sẽ bị xoá.
   Merge 3 chiều thường tự giữ chúng (phía đích đổi, phía kia không đổi ⇒ git lấy phía đích).
2. **"0 conflict" không phải bằng chứng làm đúng** — với `-s ours` nó là bằng chứng không câu
   hỏi nào được trả lời, kể cả file chưa ai nhìn.
3. Quy trình/công cụ có tài liệu mà mình nghi hỏng ⇒ mô phỏng đúng quy trình đó
   (`git merge-tree`) TRƯỚC khi kết luận: cách dùng sai của mình trông y hệt một bug.

#### Sửa file bằng SCRIPT — ba ràng buộc

Áp cho **MỌI thao tác script đọc rồi ghi đè file** — resolve conflict chỉ là một trường hợp
(patch hàng loạt, đổi tên, chèn khai báo cũng cùng cơ chế hỏng). Sửa tay thì bỏ qua mục này.

1. **Ghi lại đúng trạng thái byte của TỪNG file.** Đọc binary, ghi nhận BOM + kiểu EOL của
   từng file rồi ghi lại y như vậy; **không áp một encoding chung cho cả lô**. Python:
   `open(..., newline='')`, encoding `utf-8` — **KHÔNG** dùng `utf-8-sig` cho cả lô: nó ăn mất
   BOM của file có và (khi ghi) thêm BOM vào file vốn không có — thay đổi không ai yêu cầu.
2. **Đừng chèn `\n` thô vào file toàn CRLF** — sinh ra một dòng lone-LF lẫn vào giữa.
3. **Verify bằng ĐẾM BYTE:**

```python
b = open(path, 'rb').read()
print('CRLF:', b.count(b'\r\n'), 'LF tong:', b.count(b'\n'), 'BOM:', b[:3] == b'\xef\xbb\xbf')
# CRLF == LF tong  -> thuan CRLF.   Lech nhau -> co dong lone-LF.
```

| Công cụ | Vì sao KHÔNG dùng để kết luận |
|---|---|
| `file <path>` | báo "CRLF, LF line terminators" — mơ hồ, không cho biết bao nhiêu |
| `grep -c $'\r'` / `awk '/\r$/'` | trong Git Bash trên Windows hai lệnh cho kết quả **mâu thuẫn nhau** (một nói mọi dòng có CR, một nói không dòng nào có) |
| `git diff` | ⚠ với `core.autocrlf=true` git **normalize EOL lúc so sánh** nên lệch EOL KHÔNG hiện ra — lọt thẳng vào commit |

> **Nguyên tắc**: một bước verify chỉ có giá trị khi công cụ verify đo đúng thứ có thể hỏng.
> Với thuộc tính ở tầng byte (EOL, BOM, encoding) mọi công cụ line-oriented đều là suy đoán,
> và bất kỳ lớp normalize nằm giữa cũng làm bước verify mù đúng chỗ nó cần sáng nhất.

#### File cố ý KHÔNG commit (config máy dev) khi phải làm sạch cây để merge

Thứ cần giữ là **thay đổi của mình**, không phải **trạng thái của mình** — phục hồi nguyên file
backup sẽ đè im lặng thay đổi mà nhánh kia vừa đưa vào chính file đó, và vì file không bao giờ
được commit nên không có diff nào để ai phát hiện.

```bash
git diff -- <file> > <scratch>/my-local.patch   # 1. luu DIFF cua minh, khong luu ca file
git checkout -- <file>                          #    roi moi lam sach cay va merge
git apply <scratch>/my-local.patch              # 2. ghep lai len ban moi cua nhanh kia
git diff --stat -- <file>                       # 3. so dong +/- phai DUNG BANG phan cua minh
```

Patch không áp được ⇒ sửa có chủ đích, đừng chép đè. Buộc phải backup cả file ⇒ trước khi phục
hồi, `diff` HAI chiều với bản mới và giải trình từng dòng chỉ-có-ở-bản-mới.

#### Probe chẩn đoán trong thư mục state của công cụ khác

Khi chẩn đoán lỗi git (`fatal: couldn't set refs/heads/...`) mà phải tạo file bên trong
`.git/` — hoặc bất kỳ thư mục lock/cache của công cụ khác:

1. **Tên KHÔNG trùng quy ước của công cụ đó.** Hậu tố lạ như `.probe-tmp`; **không bao giờ**
   dùng `.lock`. Một `<ref>.lock` nằm lại làm MỌI lệnh git sau đó chết với "Another git
   process seems to be running in this repository" — kể cả GUI user đang mở ở terminal khác,
   nên user sẽ báo lỗi mà không biết nó từ đâu ra.
2. **Xoá trong CÙNG một lệnh shell với lệnh tạo** (`&&` / `;` / `trap`):

```bash
cp "$ref" "$ref.probe-tmp" && { mv "$ref.probe-tmp" "$ref" || true; rm -f "$ref.probe-tmp"; }
```

> **Nguyên tắc**: một phép thử chẩn đoán phải khôi phục trạng thái ngay cả khi — nhất là khi
> — nó thất bại, vì thất bại chính là kết quả ta đang mong đợi. Và tên artefact tạm phải nằm
> NGOÀI không gian tên mà công cụ chủ quản diễn giải, nếu không rác chẩn đoán sẽ được đọc
> thành tín hiệu điều khiển.

## Triggers & Procedures

### Chuỗi mặc định giữa các trigger

Các trigger của họ này là MỘT quy trình, không phải các lệnh rời. Xong một mắt thì mặc định là
**đi tiếp**, trừ khi mắt kế tiếp là một cổng hỏi:

```
create branch → code → review the last change → fix all issues → [commit] → [push] → [create MR]
   → review the MR !N (gitlab-review) → [post review] → fix all issues → [commit] → [push] → [merge]
```

- Mắt **không ghi gì ra ngoài máy local** (code, review — kể cả `review the MR` chỉ đọc remote
  —, fix trong working tree, đo/build/test) ⇒ CHẠY LUÔN, không báo-rồi-chờ.
- Mắt trong `[...]` là **cổng hỏi bắt buộc** — giữ nguyên, không vì chuỗi mà bỏ: commit mà chuỗi
  tự đi tới (kể cả commit sau fix — user tự gõ `commit and push` thì chính đó là xác nhận), push,
  mọi thứ ghi ra remote (tạo MR, post comment), merge. Hai cổng liền nhau được gộp chung một câu
  hỏi ("push và tạo MR?").
- User có quy trình khác ⇒ họ ghi đè mục này một lần, không phải nhắc lại từng bước mỗi ticket.

### "create branch <name>" hoặc "create branch from task <TASK-ID>..."

**Step 1 — Detect input mode** (parse phần text sau `create branch ...`):

| Input pattern | Mode | Hành động |
|---|---|---|
| Có prefix branch type + slug, vd `feature/HNCW-313-Bao-cao-ngay-nvkd` | **A — Full branch** | Dùng **nguyên si**, KHÔNG đề xuất, KHÔNG sửa (kể cả nếu input violate convention — chỉ warn) |
| Slug kebab-case không prefix, vd `HNCW-313-Bao-cao-ngay-nvkd` | **B — Pre-formatted slug** | Auto thêm `feature/` (convention nội bộ chỉ dùng `feature/`). **KHÔNG** bóc tách lại |
| Raw Jira title (có dấu / space / `[...]` / `(...)`), vd `HNCW-313 [Vận hành] Tạo báo cáo ngày cho NVKD(IT-10212)` | **C — Raw title** | Bóc tách → đề xuất 1-2 candidate → hỏi user pick |
| Chỉ TASK-ID, vd `HNCW-313` | **D — Bare ID** | Hỏi user description ngắn (2-4 từ) |

**Technical detection** — phần text sau `<TASK-ID>`:
- Match `^-[A-Za-z0-9-]+$` (gạch đầu, alphanumeric + gạch nối, không space/dấu) → **Mode B**
- Match `^/[A-Za-z0-9-/]+$` với prefix `feature|bugfix|hotfix/` → **Mode A**
- Có space / dấu tiếng Việt / `[`, `(`, ... → **Mode C**
- Trống → **Mode D**

> **NGUYÊN TẮC**: Mode A và B = user đã chủ động format → **respect tuyệt đối**, không tự sinh khác. Mode C và D mới được phép bóc tách + đề xuất.

**Step 2 — Bóc tách** (chỉ Mode C):

- Tách `TASK-ID` (pattern `[A-Z][A-Z0-9]+-\d+`)
- **Branch type: luôn `feature/`** — bất kể task là feature, bug fix, hay hotfix. Convention nội bộ chỉ dùng 1 prefix. Chỉ tạo `bugfix/` hoặc `hotfix/` khi user **chủ động gõ rõ** prefix đó trong Mode A (vd `create branch from task bugfix/HNCW-311-Duplicate-survey-log`).
- Bỏ scope marker đầu title (`[Supermarket - AU]`, `[Mobile]`...)
- Bỏ reference ticket khác (`(IT-12468)`, `(linked WRA-9)`)
- Drop type filler (xem rule mục Branch naming)
- **KEEP direction marker** (`Cho-phep`, `Allow`, `Validate`, `Block`, `Disallow`, `Restrict`, `Enforce`)
- **Phân loại ticket TRƯỚC khi rút từ key**:
  - Chủ đề đơn → lấy 2-4 từ key: **direction + action + phạm vi**
  - **Ô-dù** (title kiểu "cải tiến/rà soát nhiều tính năng", hoặc user xác nhận nhiều sub-item) → đặt tên theo **MẢNG bao trùm** (`Giao-dien`, `Tai-san`, `Bao-cao`), không cố nêu một tính năng, và không giữ lại `Cai-tien` để lấp chỗ. Lấy mảng từ tiền lệ thật: nhánh của 2-3 ticket cùng loại (`git branch -r`) + scope commit của chúng
  - Sau khi drop còn **0-1 từ key** (hoặc chỉ còn tên dự án/repo, danh từ chung "tính năng", "hệ thống") và không có mảng chung → **STOP, không đề xuất** (xem Edge case)
- Đo quy ước tiền tố thật của repo (khối ⚠️ dưới bảng `Branch naming`) trước khi dựng tên

**Step 3 — Đề xuất** (Mode C, D):

- Đưa 1-2 candidate kèm length character count. Hai candidate phải khác nhau về **TRỤC** (cách đặt tên — kể tên đối tượng vs gọi theo mảng), không phải về độ dài của cùng một khung: một khung in hai lần vẫn là một lựa chọn
- **Self-check trước khi in candidate** (candidate do skill sinh ra phải qua cùng bộ luật skill dùng để phê phán input — nhất là ở lượt sinh lại thứ hai, khi chú ý dồn sang trục mới):
  1. Đối chiếu từng từ với danh sách **type filler** ở bảng `Branch naming` — có ⇒ xoá, sinh lại
  2. Sentence case strict (mọi từ sau từ đầu lowercase, kể cả viết tắt)
  3. 2-4 từ key, tổng ≤ 50 ký tự
  4. Chỉ khi 3 bước trên sạch mới in
- **DỪNG, hỏi user pick option nào** (hoặc override description bằng tên user tự gõ)
- User trả lời bằng một **DỮ KIỆN về task** ("task này cải tiến nhiều tính năng") thay vì chọn ⇒ KHÔNG hỏi lại cùng bộ option: dữ kiện đó là tên của trục bị thiếu — dựng lại option theo trục đó rồi mới hỏi lần hai
- **KHÔNG được tự tạo branch** trước khi user xác nhận. Tránh tình huống user phải rename sau

**Step 4 — Tạo branch** (mọi mode):

1. Đảm bảo working tree sạch (`git status`); có thay đổi chưa commit → hỏi user trước khi tiếp tục
2. Xác định `<base>` theo mục **Base branch**: trừ khi user đã nói rõ trong prompt, **HỎI user chọn TÊN base branch (`main` hay `dev`)** (chạy `git branch -r` trước để gợi ý đúng tên nhánh thật, đã lọc nhánh deploy). Câu hỏi chỉ chọn tên nhánh — **KHÔNG** đưa option "không pull"/"local sẵn có". Sau khi chọn, **LUÔN** lấy code mới nhất rồi mới tạo branch: `git fetch origin <base> && git checkout <base> && git pull` (áp dụng cho mọi base, kể cả `dev`)
   - **Dò TASK-ID đã dùng chưa** — ở nơi định danh thực sự sống (remote + history), không chỉ working tree. Gộp chung message với `git status`:
     ```bash
     git branch -a --list "*<TASK-ID>*"
     git log --oneline --all --grep="<TASK-ID>" | head -5
     ```
     Có hit ⇒ báo MỘT dòng (nhánh/commit trùng) rồi **hỏi trước khi tạo**: (a) ID gõ nhầm → xin ID đúng; (b) task cũ cùng ID → làm tiếp nhánh cũ hay tạo nhánh song song. Trùng có thể hợp lệ (Jira tái dùng số) — quyết định là của user, nhưng không được tạo im lặng: ID sai sẽ nhân bản vào mọi commit, MR title, footer
3. Tạo branch (luôn nhánh `<base>` đang checkout):
   - Mode A/B: `git checkout -b <input-nguyên-si>` (Mode B: thêm prefix `feature/` mặc định)
   - Mode C/D: `git checkout -b <branch-user-pick>` (chỉ sau khi user đã chọn ở Step 3)
4. Báo lại tên branch + length character count

**Edge case**:

| Tình huống | Xử lý |
|---|---|
| Mode A/B branch >50 chars | Warn user nhưng **KHÔNG ép sửa** — user đã chủ động chọn |
| Mode C sau khi trim vẫn >50 chars | Đề xuất viết tắt (`qty` thay `so-luong`, `co` thay `checkout`) hoặc bỏ phạm vi |
| Mode C sau khi drop filler còn **0-1 từ key**, hoặc chỉ còn tên dự án/repo/danh từ chung | **STOP, KHÔNG đề xuất candidate.** Hỏi user 1 câu task thật sự làm gì. 🚫 Cấm giữ lại filler vừa drop để lấp, cấm suy nội dung từ tên dự án. "Không đủ từ key" là kết quả hợp lệ. Title rỗng nghĩa hay đi cùng ID gõ nhầm/tái dùng ⇒ chạy luôn phép dò TASK-ID ở Step 4 |
| TASK-ID đã xuất hiện ở nhánh remote/local hoặc `git log --all` | Báo nhánh/commit trùng, **hỏi user trước khi tạo** (Step 4.2) |
| TASK-ID không match pattern `[A-Z][A-Z0-9]+-\d+` | STOP, hỏi user |
| Cần branch type khác `feature/` | User phải **gõ rõ prefix** trong input, vd `create branch from task bugfix/HNCW-311-Duplicate-survey-log` (Mode A — skill dùng nguyên si). Skill **KHÔNG** tự suy đoán `bugfix/`/`hotfix/` từ nội dung task |
| User muốn đổi tên branch sau khi skill đã tạo | Dùng trigger riêng `rename branch <new-name>` (xem mục bên dưới). Không tự rename bằng `git branch -m` mà không update upstream → sẽ phá `commit and push` |

### "rename branch <new-name>" hoặc "rename branch sang <new-name>"

User không thích tên branch skill vừa tạo và muốn đổi. Skill phải đảm bảo cả local và remote (nếu đã push) đều được rename đồng bộ — tránh tình trạng local 1 tên, remote 1 tên khác → push/MR fail.

**Step 1 — Detect trạng thái**:

```bash
git branch --show-current                              # tên local hiện tại
git rev-parse --abbrev-ref --symbolic-full-name @{u} 2>/dev/null  # upstream (nếu có)
```

| Trạng thái | Hành động |
|---|---|
| Branch chưa push (chưa có upstream) | Rename local thuần: `git branch -m <new-name>`. Xong, không cần đụng remote |
| Branch đã push (có upstream) | Cần rename cả 2 phía (Step 2-3) |

**Step 2 — Rename local + push tên mới**:

```bash
git branch -m <new-name>
git push -u origin <new-name>
```

**Step 3 — Xóa branch cũ trên remote**:

Hỏi user: "Branch cũ `<old-name>` còn tồn tại trên remote. Xóa không?"
- Yes → `git push origin --delete <old-name>`
- No → giữ lại (nhưng warn: 2 remote branch trỏ cùng commit, có thể confuse reviewer)

**Step 4 — Verify**:
```bash
git branch -vv                  # xem local + upstream mới
git ls-remote --heads origin    # check remote không còn old-name (nếu đã xóa)
```

**Lưu ý**:
- KHÔNG dùng `git branch -m` thuần khi branch đã push — sẽ break upstream tracking
- Nếu đã có MR mở trên branch cũ: rename remote sẽ làm MR đứng (URL không đổi nhưng source branch không tồn tại). Phải đóng MR cũ + tạo MR mới với branch mới, hoặc dùng `glab mr update <N> --source-branch <new-name>` nếu glab support

### "merge main vào nhánh feature" / "gỡ conflict" / "merge main vào bị conflict"

Ca thường gặp nhất trong ngày, và là ca **duy nhất** trong họ skill này có đích merge là
nhánh feature đang mở MR — không phải nhánh deploy.

> Phân biệt với hai skill kia: `gitlab-sync` lo `main → builds/dev/<app>`,
> `gitlab-cherrypick` lo `main → release/<app>/<v>`. Cả hai resolve trên nhánh TRUNG GIAN
> rồi PR ngược. Ca này resolve **thẳng trên nhánh feature**, vì nhánh feature vốn đã là của
> mình.

**Step 1 — Xác định repo và CHIỀU merge** (đừng đoán):

```bash
git rev-parse --show-toplevel     # dang o repo nao
git status                        # co dang giua merge khong
cat "$(git rev-parse --git-dir)/MERGE_MSG" 2>/dev/null   # cho biet merge tu dau vao dau
```

`MERGE_MSG` là chỗ duy nhất nói rõ chiều merge khi đã ở giữa conflict. Không đọc nó thì rất
dễ resolve ngược (giữ phía feature khi đáng giữ phía main).

**Step 2 — Phân loại từng file conflict**:

```bash
git diff --name-only --diff-filter=U
```

| Tình huống | Bên nên giữ | Lý do |
|---|---|---|
| File main sửa, feature KHÔNG chạm | **main** | Lấy nguyên thay đổi mới của main |
| File feature đang làm, main chỉ format/rename | **feature**, áp lại đổi tên của main | Giữ công việc đang làm |
| Cả hai bên đều thêm code mới vào cùng chỗ | **GỘP cả hai** | Kiểu "both added" — bỏ bên nào cũng là mất tính năng |
| Không rõ | **STOP, hỏi user** | Resolve sai trên nhánh feature = lỗi lọt vào MR |

Sửa bằng script, định lấy-trọn-một-phía, hoặc có file config cố ý không commit phải làm sạch
trước khi merge thì đọc `Conventions → Resolve conflict (luật dùng chung)` ở trên — **bắt
buộc**, không phải khuyến nghị.

**Step 3 — Verify bằng BUILD + TEST, không chỉ bằng `git status`**:

Chạy build và test của project **liên quan tới file vừa sửa** (không cần cả solution).
`git status` sạch chỉ nói marker đã xoá, không nói code còn biên dịch được.

**Step 4 — DỪNG ở `git add`, KHÔNG tự commit merge**:

```bash
git add <file>...
git status        # bao lai cho user
```

> 🚫 Không tự chạy `git commit`. Commit merge là quyết định của user — họ có thể muốn xem
> lại resolution, hoặc abort (`git merge --abort`) sau khi thấy diff. Báo cáo: file nào giữ
> bên nào, build/test đã chạy gì, rồi chờ.

### Sinh code từ mô tả task
- Khi user paste mô tả task Jira làm prompt, đọc kỹ và xác nhận lại scope trước khi code nếu có chỗ mơ hồ
- Code theo convention của project (tham khảo CLAUDE.md nếu có, hoặc đọc file gần khu vực sửa để bắt chước style)
- Không thêm tính năng/refactor ngoài scope task
- Sau khi xong, tóm tắt ngắn các file đã thay đổi

### "review the last change" / "review change" (+ optional "simplify")

Trigger match là lenient: thêm từ `simplify` bất kỳ vị trí trong câu để bật Step 0; không có thì bỏ qua Step 0.

**Step 0 — Simplify pass** (chỉ chạy khi trigger chứa `simplify`):

1. Capture uncommitted + staged diff (`git diff` và `git diff --cached`). Empty → báo skip Step 0 và sang Step 1.

2. Scan diff theo 2 lens **Efficiency** + **Quality & Reuse** — định nghĩa và danh sách flag xem mục **`Review lenses`** ở `Conventions`. Bỏ qua 2 lens `Correctness` và `Security`: đây là pass **dọn dẹp**, không phải pass tìm bug. Inline Claude, không spawn agent vì scope hẹp.

3. **Auto-fix trực tiếp** mọi finding rõ ràng — false positive thì skip, không cãi, không hỏi user từng issue. Fix độc lập ở các file khác nhau → batch parallel trong 1 message.

4. Báo tóm tắt số issue đã fix + file đã đụng (hoặc "code đã sạch") rồi sang Step 1. **KHÔNG tự commit** — fix nằm ở working tree, gộp chung với review issues user fix sau.

**Step 1 — Capture diff + NẠP CONTEXT** (đừng review diff trong "ống hút"):

1. Capture diff: `git diff` (hoặc `git diff HEAD` nếu đã staged). Trong simplify mode, đây là diff sau-fix.
2. **Đọc FULL các file đã đổi** (không chỉ diff hunk) — để thấy code xung quanh, import, hàm gọi tới. Review chỉ-diff là nguyên nhân #1 gây finding sai (đoán những thứ không nhìn thấy).
3. **Grounding convention**: đọc `CLAUDE.md` (nếu có) + 1-2 file lân cận cùng thư mục/module để học convention THẬT của repo — đừng áp convention generic.
4. **Grounding task**: nếu mô tả task (Jira) đã có trong hội thoại → dùng làm chuẩn "logic đúng/đủ chưa". Nếu CHƯA có và định đánh giá logic/edge-case → hỏi user 1 câu ngắn về mục tiêu task, hoặc nói rõ "review này chỉ xét quality/efficiency, không phán logic vì thiếu spec".
5. **Grounding NGOÀI diff** — diff chỉ cho thấy phần đã đổi; các lỗi dưới đây nằm ở chỗ diff không chạm:
   - a. Hàm/API/schema đổi signature hoặc behavior ⇒ grep **mọi caller**, kể cả file ngoài diff. Field/cột bị gỡ khỏi mọi đường GHI mà vẫn còn trong model/DB (diff chỉ có dòng `-` cho field đó) ⇒ grep **tên field trần** toàn repo để liệt kê bên ĐỌC — bên đọc không phải caller của hàm nào nên grep chữ ký không thấy.
   - b. Kết luận **PHỦ ĐỊNH** ("không caller nào", "không có luật", "chưa ai dùng") ⇒ xác nhận bằng phép quét thứ hai rộng hơn hẳn (bỏ mọi bộ lọc đường dẫn), và ghi phạm vi đã quét vào chính kết luận. Kết quả MỘT PHẦN (lệnh vẫn ra vài dòng) nguy hiểm hơn kết quả rỗng: nó tắt mọi nghi ngờ.
   - c. Caller ở **REPO KHÁC** ⇒ phân giải nhánh/MR đồng hành trước khi kết luận "phía kia chưa sửa": `glab mr list` ở repo đó tìm cùng TASK-ID/tên nhánh → không có thì `git ls-remote --heads origin | grep <TASK-ID>` → có thì fetch vào ref riêng và đọc ở đó, **không** đọc working tree của repo đó (nó đang đứng ở nhánh ngẫu nhiên). Đã có bản sửa ở MR đồng hành ⇒ `Blocker` → `Major` với bản chất khác: **ràng buộc thứ tự deploy**.
   - d. MR dựng từ **MỘT MẪU**:
     - file tự khai "mirror/port/copy từ X" ⇒ với từng cặp hàm tương ứng, diff **GUARD** (early-return, `await` trước khi build payload, cờ đang-xử-lý, `?.`/`??`, điều kiện trước `setState`), không diff toàn file; đọc chú thích quanh guard ở X — nó thường ghi cơ chế lỗi đã gặp, dùng làm bằng chứng severity. Độ trùng thấp giữa hai file KHÔNG loại trừ guard bị rơi.
     - bám **KHUÔN** từ repo/file khác ⇒ review cả khuôn: chỗ khuôn có mà bản này thiếu, chỗ bản thân khuôn sai ("đang chạy production" không phải "đã kiểm"), và chi phí của cùng một dòng có đổi bậc ở repo đích không.
     - thêm **thành viên thứ N+1** vào một họ cùng khuôn (app/module/service ngang hàng) ⇒ lấy **HIỆU tập file** với từng anh em: file có ở ≥2 anh em mà thiếu ở thành viên mới là finding cho tới khi chứng minh không cần. Trong config chép sang, field **định danh** (port, base URL nội bộ, AppId, audience, tên DB/queue) TRÙNG anh em là lỗi, không phải "đúng khuôn". Báo cáo ghi rõ đã chạy phép lấy hiệu — "không thiếu gì" và "chưa kiểm" trông y hệt nhau.
   - e. Chú thích biện minh / viện dẫn là **LỜI KHAI**, không phải kết luận:
     - viện dẫn một artefact làm nguồn ("theo đúng X.cs", "cùng logic với Y") ⇒ phân giải bằng `git ls-tree -r --name-only` / `git grep` tại đúng revision: có → so từng điều kiện; không có trong repo → đối chiếu với truy vấn/hàm tương đương GẦN NHẤT đang chạy; có mà không chứa điều được viện dẫn → finding.
     - biện minh liệt kê N mục đích ⇒ tick lý do với TỪNG mục (lý do phủ k/N ⇒ finding ở phần còn lại); lý do không nêu phạm vi (chế độ, nhánh) mà code path dùng chung ⇒ kiểm từng chế độ; chú thích "ở đây không cần … như <module khác>" ⇒ đi đọc module đó.

> 🔗 Bước 2-3 và 5 ở trên overlap với **Block 2 (Grounding)** trong prompt 4-agent ở `gitlab-review` — **không phải bản sao y hệt**: bước 5 (a–e) ở đây và bước 3–7 của Block 2 là CÙNG một danh sách, khối "Phủ phần CÒN LẠI" ở Step 3 dưới đây ↔ bước 8 của Block 2 (Block 2 chép dạng rút gọn vì subagent không đọc được file này); Block 2 có thêm bước 9 đọc **mặc định của framework** (chỉ review sâu mới cần); Step 1 có thêm **Grounding task** (bước 4) mà Block 2 không có. Sửa phần chung thì sửa cả hai bên — đừng gộp thành 1 quy tắc.

**Step 2 — Review** theo các tiêu chí (chỉ flag khi đã đọc đủ context ở Step 1):
- Logic đúng với mô tả task không *(chỉ phán khi có task context — xem Step 1.4)*
- Có edge case nào chưa cover không
- Có vi phạm convention/coding standard không *(theo convention thật đã đọc, không generic)*
- Có code thừa, dead code, hoặc abstraction không cần thiết
- Có lỗ hổng bảo mật (input validation, auth bypass, injection) không
- Có ảnh hưởng performance đáng kể không
- Diff khớp tình huống nào ở **Flag kích hoạt theo tình huống** (dưới bảng `Review lenses`) ⇒ chạy đúng flag đó

**Step 3 — Verify findings (lọc false positive TRƯỚC khi báo)**: với mỗi finding, tự kiểm:
- Gắn được **`file:line` cụ thể** không? Không → bỏ.
- **Chứng minh được bằng code đã đọc** (không phải suy diễn từ diff) không? Không chắc → bỏ hoặc hạ thành "cần xác nhận", đừng list như lỗi chắc chắn. Bằng chứng chỉ gián tiếp thì "cần xác nhận" là đúng chỗ — nhưng thử **đường đo** dưới đây trước khi hạ.
- Đề xuất fix có **thật sự áp dụng được** với codebase này không (helper/util mình gợi ý có tồn tại không)? → verify rồi mới đề xuất.

**Đường đo — biến nghi ngờ thành bằng chứng** (chọn theo loại finding):

| Loại finding | Phép đo |
|---|---|
| Ngữ nghĩa lúc chạy của thư viện bên thứ ba (ném hay nuốt, trả template hay giá trị) | Đọc mã thư viện chỉ ra GIẢ THUYẾT. Import và **chạy** chính hàm đó với dữ liệu thật tại đúng revision. Đã chứng minh một ca ⇒ quét cả tập cùng loại bằng vòng lặp ("1/26 chỗ", không "có một chỗ"); bản vá đề xuất cũng chạy qua giàn đó |
| Thư viện ĐÓNG (DLL, không có source) | Tìm **tiền lệ cùng hình dạng** đã chạy production trong chính cặp class/hàm đó, có từ trước MR (cùng lời gọi, cùng cấu hình). Có ⇒ bác, ghi tiền lệ làm bằng chứng. Không có ⇒ mới chạy thật hoặc "cần xác nhận" |
| Giá trị từ nguồn A đưa vào phép so ở đích B (cột/bảng/API khác, dù cùng tên) | "Cùng miền giá trị" là giả thuyết. Không chạm được dữ liệu ⇒ tìm **hằng số/enum/mapping đã có trong repo cho B**; một giá trị lệch là đủ báo `Major` kèm trích dẫn |
| "Thiếu test / thiếu ca kiểm" | **Viết** đúng test đó vào file tạm, chạy, xoá trong cùng một lệnh. Pass ⇒ `Minor` (lỗ test, kèm test chạy được vào đề xuất); fail ⇒ `Blocker`/`Major` + đã có repro; thiếu fixture ⇒ nói rõ chỗ thiếu thay vì đề xuất chung chung |
| Tiền đề "X luôn thoả P" (của finding, hoặc của lý do bác nó) | P được **code** cưỡng chế (validator, constraint DB, không tồn tại đường ghi, kiểu) hay chỉ **tài liệu/quy ước** ghi? Chỉ code mới cho nói "không thể xảy ra" — trỏ `file:line` chỗ chặn; trích được câu văn trong tài liệu không phải là đã kiểm |
| Fix "làm lỗi lộ sớm/ồn ào" (validate, assert, throw), hoặc dòng tự nhận là chốt chặn | Dựng **đúng điều kiện hỏng** rồi ghi: (a) tiến trình còn sống không, (b) request trả gì, (c) tầng giám sát nào bắt và sau bao lâu. `throw` là chuyện ngôn ngữ, "chết" là chuyện runtime; chốt chặn phải được cho ăn đúng ca nó nói là chặn |
| Đúng/sai phụ thuộc **thứ tự khởi tạo/đăng ký** của framework (DI, middleware, handler chain, hook) | Đọc code và tài liệu đều không đủ. Dựng mẫu thử tối thiểu với đúng phiên bản thư viện, đo một đại lượng quan sát được (số request tới nơi, số lần handler chạy). Tài liệu có "earlier/before/first" ⇒ hỏi *trước cái gì* — lúc đăng ký hay lúc dựng |

Mọi giàn thử dựng tay phải có ≥1 **ca ĐỐI CHỨNG**: ca bắt buộc chạy được (hoặc, khi dùng để BÁC
finding, ca bắt buộc phải đỏ nếu code sai). Ca nghi ngờ và ca đối chứng cùng hỏng = giàn thử sai
— sửa giàn, không báo finding.

**Dựng ca tái hiện** cho giá trị ghép/so/dẫn xuất từ ≥2 nguồn:
- Mọi nguồn mang giá trị **khác nhau và khác mặc định** — hai đầu vào cùng derive từ một `now`/fixture/record thì ca đó chỉ chứng minh "x ghép với x ra x".
- Bảng ca = **tích Descartes** các mốc biên của TỪNG nguồn (không ô nào rỗng), quét cả hai chiều lệch. Bảng ca do chính giả thuyết sinh ra thì không bác được giả thuyết.
- Tự dự đoán kết quả TRƯỚC khi chạy; đề xuất bản vá thì chạy lại đúng bảng đó trên bản vá.

**Phủ phần CÒN LẠI, đo đúng tầng**:
- MR **dọn dẹp / sửa hàng loạt** (xoá code chết, đổi tên, sửa chú thích) ⇒ đối chiếu từng khẳng định của tác giả là phép kiểm YẾU — nó không thấy thứ nằm ngoài danh sách tác giả đưa ra. Dùng phép kiểm tự **sinh danh sách** từ repo tại đúng revision (mọi tham chiếu file, mọi khoá, mọi import) rồi phân giải từng cái; soi kỹ dòng KỀ dòng đã sửa.
- Ngược lại, finding "sửa chỗ này còn sót chỗ kia" ⇒ đo ở **tầng bản sửa thực sự nằm**: theo luồng gọi từng ứng viên xuống điểm thực hiện hành vi; ứng viên hội tụ vào leaf đã vá ⇒ loại. Grep theo hình dạng đếm ĐIỂM GỌI, còn bản sửa ở leaf tác động lên ĐƯỜNG ĐI.
- "Đã chặn giá trị xấu" ⇒ liệt kê mọi **đường RA** của giá trị (render, xuất file, clipboard, tham số URL, log, payload gửi lên server). Đường không gọi hàm chặn là đường thứ hai, không phải đường an toàn — mỗi đường một phép kiểm.

**Hạ hoặc bác một finding cũng phải qua verify**:
- Hạ bằng phép đo ⇒ liệt kê TRƯỚC các nguồn dữ liệu finding phủ (câu finding có số nhiều / chữ "và" = nhiều nguồn), đo TỪNG nguồn; chưa đủ ⇒ giữ severity, ghi "đã đo A, chưa đo B".
- Finding "nới lỏng chốt X, còn chốt Y phía sau" ⇒ đo Y có **đang BẬT**: grep cờ vô hiệu hoá (`DryRun`, `Enabled`, `Disabled`, `Bypass`, `AllowAll`) trong cấu hình của đúng app, kể cả file ngoài diff. Không đo được ⇒ ghi "chưa xác nhận tầng sau có bật", không mặc định là bật; tầng sau tắt ⇒ severity lên mức "không còn lớp nào chặn".
- Chạy lại phép đo của finding ra số khác ⇒ chứng cứ sai mới chỉ hạ CHỨNG CỨ, chưa chạm KẾT LUẬN. Tự dựng phép đo cho mệnh đề và phán theo kết quả của mình; bác vì "đếm sai" là bác người đưa tin, không phải bác tin.
- Lý lẽ "hiện trạng repo", "đã có tiền lệ", "vai của finding đã bị bác" ⇒ luật ở `Review output`.

**Build/test fail khi đang review KHÔNG tự động là finding.** Phân loại theo MÃ lỗi, không theo
dòng tổng kết (`Build FAILED`, `N Error(s)` gộp mọi nguyên nhân về một hình dạng): lỗi code
(`CS*`, `TS*`, assertion) → finding; lỗi môi trường (file bị khoá `MSB3021`/`MSB3027`/`EBUSY`,
port bận, thiếu secret, output path dùng chung) → gỡ nhiễu rồi đo lại, báo theo lần đo sạch và
nói đã gỡ gì. Cách gỡ nhiễu sinh ra lỗi MỚI là lỗi của CÁCH ĐO, không phải của code. Không phân
loại được → "cần xác nhận".

> 🔗 Bước này overlap với tiêu chí verify Blocker/Major ở **Phase 2.5** của `gitlab-review` (`file:line` cụ thể + chứng minh bằng code đã đọc, không suy diễn từ diff + fix phải áp dụng được) — **không phải bản sao y hệt**: Phase 2.5 giữ bản TÓM TẮT của đường đo / ca tái hiện / hạ-bác-phải-verify ở trên và trỏ về đây; nó có thêm kiểm **reachability**, phân loại **trong/ngoài diff**, và xử lý **hai lens chữa ngược nhau** mà Step 3 không có, vì input của nó là 4 agent song song. Sửa phần chung thì sửa cả hai bên.

> Thà báo 3 issue **chắc** còn hơn 10 issue nửa đoán. Finding không qua được Step 3 thì **không đưa vào danh sách**.

**Step 4 — Báo cáo** dưới dạng danh sách có đánh số, **severity đứng ngay sau số** để user lọc nhanh:

```
#1 [Blocker] path/file.cs:42 — <vấn đề>. Đề xuất: <fix>
#2 [Major]   path/file.cs:88 — <vấn đề>. Đề xuất: <fix>

### Minor (không fix — user tự quyết)
#3 [Minor]   path/file.cs:15 — <vấn đề>
```

Không có Blocker/Major → báo "Không có vấn đề chặn" rồi dừng.

**Tự soát chính báo cáo trước khi đưa ra** (dùng chung cho mọi báo cáo review của họ này):
- Finding kèm số liệu ⇒ câu điều kiện mình viết **dự đoán** con số nào? So với số đã đo; lệch ⇒ một trong hai sai, thường là câu chữ. Viết "độc lập với X" thì phải có phép quét theo X.
- Con số `×N` của công cụ quét tĩnh là số **khớp mẫu** (cận trên), không phải số lỗi — mở từng chỗ rồi báo "N khớp mẫu, M xác nhận".
- Trạng thái build/test/typecheck chỉ là bằng chứng khi **đầu vào không rỗng**: đếm file/test thực chạy và so với file của diff (vd `tsc --listFilesOnly | wc -l`; tsconfig dạng solution `files: []` + `references` cần `tsc -b`). Exit 0 trên tập rỗng xanh y như xanh thật — kể cả khi đó là lời khai "đã typecheck" của tác giả MR.
- Sắp viết "chưa làm / không làm được X" ⇒ kiểm công cụ có thật sự thiếu không, gồm cả tool **nạp trễ** (tên có trong danh sách nhưng schema chưa nạp — phải `ToolSearch` trước khi gọi). Chỉ viết kèm lý do cụ thể: không có tool, bị từ chối quyền, môi trường thiếu.

**Lưu ý — chọn đúng độ sâu (đừng kỳ vọng sai vào công cụ nhẹ)**:
- Trigger này cố tình **lightweight** (inline, không spawn agent) → hợp để **liếc nhanh** đoạn vừa sửa. Dù đã nạp context + verify, nó vẫn nông hơn review chuyên sâu.
- Muốn **chính xác/sâu hơn**: dùng skill built-in **`/code-review`**, hoặc trigger **`review the whole branch`** (skill **`gitlab-review`** — chỉ Lead cài; 4 agent chuyên biệt đọc full file + tầng verify).
- Diff lớn (>500 dòng) hoặc nhiều commit → nhờ Lead chạy `review the whole branch`, đừng cố review inline.

### "Commit and push"

Spec đầy đủ Conventional Commits + Jira ID + push gate. Self-contained: không cần cài skill `commit` riêng.

> **Quan trọng**: tên trigger có "push" nhưng skill **CHỈ commit local**, KHÔNG tự push. Push là hành động remote → bắt buộc hỏi user xác nhận.

**Trigger phụ**: thêm `--quick` ("commit and push --quick", "quick commit") → kích Quick mode (xem `## Quick mode` ở [`commit-reference.md`](commit-reference.md)).

#### Inputs

| Input | Rule |
|---|---|
| TASK-ID | Auto-extract từ tên nhánh hiện tại (`feature/WRA-9-...` → `WRA-9`). Pattern `[A-Z][A-Z0-9]+-\d+`. Không match → STOP, hỏi user |
| Repo language | Tiếng Việt (theo `git log`) — áp dụng cho `subject` và `body` |
| Detect "quick" intent | User nói "nhanh" / "quick" / "tạm" / "small" / "fast" → suggest `--quick` trước khi commit |

#### Behavior

| Rule | Detail |
|---|---|
| Probe trước khi quyết định | Luôn chạy Step 1 đầy đủ — không skip kể cả commit nhỏ |
| Không bao giờ guess `type`/`scope` | Không chắc → STOP, hỏi user. Không coin-flip |
| Quality > speed | 1 câu hỏi xác nhận đỡ 1 commit sai format |

#### Process

**Step 1 — Probe repo state** (parallel calls trong 1 message):

| Call | Mục đích |
|---|---|
| `git status` (không `-uall`) | Untracked + modified files |
| `git diff --cached` | Staged hunks only |
| `git diff` | Unstaged hunks only — tách biệt để detect partial-staging |
| `cat "$(git rev-parse --show-toplevel)/.commit-scopes"` | Scope allowlist (works từ subdir) |

- **Đọc dòng `On branch X` của CHÍNH lần chạy này**, không dùng tên nhánh nhớ từ lượt trước — repo là tài nguyên dùng chung với user, họ có thể đã checkout nhánh khác (vd nhánh deploy để build) giữa hai lượt. Cổng chặn cơ học nằm ở Step 5.
- **Diff có FILE MỚI** (`??` / `A` trong `git status`) ⇒ chạy **gate đầy đủ** của project (typecheck + lint + test + build) trước khi commit. Test runner (vitest/jest…) phần lớn transpile rồi vứt kiểu, không type-check: "test của file đó xanh" không chứng minh file biên dịch được, mà file mới là lần đầu nó đi qua trình biên dịch.

**Step 2 — Partial-staging guard**:

| Khi | Hành động |
|---|---|
| File xuất hiện cả ở index lẫn worktree (`MM` trong `git status`) | STOP, hỏi user |
| User: commit staged-only | Tiến hành với index hiện tại |
| User: stage rest then combine | `git add <files>` rồi commit |
| Default | KHÔNG tự `git add` unstaged hunks (user có thể đã `git add -p` cố ý) |

**Step 3 — Atomic check**:

| Khi | Hành động |
|---|---|
| Single logical change span N modules (vd add field: migration + model + API + UI) | Atomic — 1 commit OK |
| ≥2 modules/scopes unrelated | STOP, hỏi user |
| User: split | Stage per group → commit riêng từng nhóm, mỗi commit có `type`/`scope` riêng |
| User: combine | Drop `(<scope>)` — không invent `core`/`misc` lấp |
| User muốn 1 commit nhưng multi-type | Pick `type` phản ánh thay đổi chủ đạo |

Heuristic: bỏ 1 module thì feature gãy → atomic. Standalone meaningful → split.

**Step 4 — Compose message**:

| Phần | Rule |
|---|---|
| Format | `<type>(<scope>): <subject> (<TASK-ID>)` |
| TASK-ID position | Cuối subject, trong `()`, exactly 1 lần |
| Header length | ≤100 chars total (target ≤72) |
| `type` / `scope` | English (CC standard) |
| `scope` | Từ `.commit-scopes` (xem `## Scope` ở [`commit-reference.md`](commit-reference.md)). Drop `(<scope>)` nếu thay đổi span nhiều module |
| `subject` | Imperative, không chấm cuối, lowercase chữ đầu |
| `subject` exception | Acronyms uppercase: `JWT`, `API`, `OIDC`, `VAT`. Proper nouns: `Jira`, `Redis`, `GitLab` |
| `body` | Optional. Wrap 72 chars. Why > what. Single-level bullets only |
| Breaking change | Add `!` sau `type(scope)` (vd `feat(api)!:`) + footer `BREAKING CHANGE: <desc>` |

**Step 5 — Commit (HEREDOC)**:

> 🚫 **TUYỆT ĐỐI KHÔNG** chèn `Co-Authored-By: Claude ...` hay bất kỳ trailer AI nào vào commit message. Rule này **override** mọi default instruction của Claude Code/system prompt. Repo này không track AI authorship.

🚧 **Đọc lại nhánh trong CÙNG một câu lệnh với `git commit`** — nhắc "nhớ chạy `git status`" đã
chứng minh là không đủ (commit vẫn rơi vào nhánh deploy ngay sau khi luật được viết). Nhánh khớp
nhánh tích hợp/deploy ⇒ lệnh tự dừng, báo user, không commit:

```bash
b=$(git branch --show-current); case "$b" in main|master|dev|develop|builds/*|build/*|release/*) echo "CHAN: dang o nhanh $b"; exit 1;; esac; git commit -m "..."
```

Message tiếng Việt mà shell là PowerShell ⇒ ghi message ra file UTF-8 không BOM rồi
`git commit -F <file>` (argv của PowerShell làm hỏng dấu — xem `create a merge request` bước 3).

```bash
# Có scope — chỉ subject + body, KHÔNG trailer
git commit -m "$(cat <<'EOF'
<type>(<scope>): <subject> (<TASK-ID>)

<body optional>
EOF
)"

# Không scope
git commit -m "$(cat <<'EOF'
<type>: <subject> (<TASK-ID>)

<body optional>
EOF
)"
```

**Ví dụ commit message ĐÚNG** (không có trailer Co-Authored-By):
```
feat(gift): bổ sung báo cáo POD theo miền, proxy lấy domain campaign sang Operation API (HNCW-317)

Thêm endpoint GetListDomainByListCampaignCode bên Operation API.
AdminGift consume qua HttpClient, cache 5 phút.
```

**Ví dụ commit message SAI** (có trailer phải xóa):
```
feat(gift): bổ sung báo cáo POD theo miền (HNCW-317)

<body>

Co-Authored-By: Claude Opus 4.7 (1M context) <noreply@anthropic.com>   ← XÓA DÒNG NÀY
```

**Quy trình self-check trước khi chạy `git commit`**:
1. Soạn message hoàn chỉnh trong head
2. Verify: subject có format `<type>(<scope>): <subject> (<TASK-ID>)` ✓
3. Verify: body (nếu có) giải thích WHY, không lặp WHAT ✓
4. Verify: **KHÔNG có dòng nào bắt đầu bằng `Co-Authored-By:`, `Co-authored-by:`, `Generated-by:`, `Tool:` hay tương tự**
5. Nếu thấy có trailer AI ở message → **XÓA** trước khi chạy `git commit`

**Step 6 — Push gate** (sau khi commit local thành công):

1. Báo commit hash + tóm tắt nội dung
2. **DỪNG, HỎI user**: "Đã commit `<hash>` ở local. Bạn có muốn push lên remote không?"
3. Đợi xác nhận rõ ràng ("ok push" / "yes" / "push đi") rồi:
4. **Detect upstream tracking trước khi push** (handle rename scenario):
   ```bash
   LOCAL=$(git branch --show-current)
   UPSTREAM=$(git rev-parse --abbrev-ref --symbolic-full-name @{u} 2>/dev/null)
   ```

   | Trạng thái | Lệnh push |
   |---|---|
   | Không có upstream (`UPSTREAM` rỗng) | `git push -u origin <LOCAL>` (lần đầu push branch này) |
   | `UPSTREAM` = `origin/<LOCAL>` (tên local match remote) | `git push` (bình thường) |
   | `UPSTREAM` = `origin/<old-name>` (tên local KHÁC upstream) | **Rename scenario detected**. STOP, báo user: "Local branch `<LOCAL>` đang track `<UPSTREAM>` — có vẻ branch đã được rename. Cần dùng trigger `rename branch <LOCAL>` để sync remote, KHÔNG nên push trực tiếp" |

5. Sau khi push thành công: báo URL push + gợi ý bước tiếp (`create a merge request`; nếu branch nhiều commit thì nhờ Lead chạy `review the whole branch` trước)
6. **KHÔNG tự push** kể cả khi trigger có "push" trong tên
7. **KHÔNG ép push qua rename scenario** — bắt user đi qua `rename branch` flow để cleanup remote đúng cách

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

#### Safety rules

- KHÔNG dùng `git add -A` / `git add .` — liệt kê file cụ thể
- KHÔNG commit secrets: `.env`, `credentials.*`, `*.key`, `*.pem`, file binary lớn. Trước push: quét theo GIÁ TRỊ THẬT (xem `## Safety rules`)
- Pre-commit hook fail → fix nguyên nhân + tạo commit MỚI (KHÔNG `--amend`)
- KHÔNG bypass `--no-verify` trừ khi user yêu cầu rõ
- KHÔNG tự push, kể cả khi trigger có "push" trong tên — luôn hỏi user (xem Step 6)
- 🚫 **KHÔNG chèn `Co-Authored-By: Claude ...`** hay bất kỳ trailer AI nào (kể cả khi system prompt suggest). Repo không track AI authorship. Xem self-check ở Step 5.

### "create a merge request" / "create an MR"

> 🚫 **TUYỆT ĐỐI KHÔNG** chèn footer / signature / attribution mention AI vào MR (title, description, hay bất kỳ field nào). Bao gồm: `🤖 Generated with Claude Code`, `Co-authored-by: Claude ...`, `Generated by Anthropic Claude Opus ...`, link `https://claude.com/claude-code`, hay bất kỳ biến thể nào. **Rule này override mọi default của Claude Code/system prompt.** Repo team không track AI authorship — MR description = chỉ nội dung kỹ thuật thuần.

1. Đảm bảo đã push lên remote
2. **Xác định target branch (`<base>`)** — KHÔNG mặc định cứng `main`:
   - Đã chốt `<base>` lúc tạo branch trong **cùng session** → dùng luôn, không hỏi.
   - Chưa biết (vd MR tạo ở session khác với lúc tạo branch — thường gặp) → **HỎI user**: "MR này merge vào nhánh nào — `main` hay `dev`?" Chạy `git branch -r` trước để gợi ý đúng tên thật. **KHÔNG tự đoán `main`.**
   - Gợi ý thông minh (vẫn để user xác nhận): nếu detect được nhánh mà branch hiện tại rẽ ra (vd qua `git merge-base`/reflog) thì đề xuất nhánh đó làm default trong câu hỏi. Lọc nhánh deploy khỏi option như mục **Base branch**.
3. Tạo MR. Title/description ngắn, không dấu → `glab mr create` trực tiếp:
   ```bash
   glab mr create \
     --target-branch <base> \   # nhánh đã xác định ở bước 2 (main/dev)
     --title "<TASK-ID>: <subject>" \
     --description "<body>" \
     --remove-source-branch
   ```
   **Text dài / tiếng Việt / markdown ⇒ KHÔNG đẩy qua argv của CLI.** Argv/pipe của PowerShell làm hỏng Unicode khi vào CLI native, và nó **hỏng theo shell chứ không theo lệnh** — cùng dòng lệnh chạy ở Git Bash thì ra đúng, nên chạy thử ở một máy không chứng minh được gì; mà thứ hỏng là artifact ĐÃ PUBLISH. Đường đã chạy thật (luật dùng chung cho cả họ — `gitlab-review`, `gitlab-sync`, `gitlab-cherrypick` giữ bản tóm tắt và trỏ về đây):
   ```bash
   # 1. mr_title.txt + mr_body.md: ghi bang tool Write (UTF-8, KHONG BOM)
   # 2. dung JSON bang python: chi truyen path TUONG DOI (interpreter Windows-native khong hieu
   #    path MSYS /c/...), json.dump mac dinh ensure_ascii -> file JSON thuan ASCII
   python -c "import json,sys; t,b=(open(f,encoding='utf-8').read().strip() for f in sys.argv[1:3]); json.dump({'source_branch':sys.argv[3],'target_branch':sys.argv[4],'title':t,'description':b,'remove_source_branch':True},open('mr.json','w'))" mr_title.txt mr_body.md <source-branch> <base>
   # 3. POST — BAT BUOC co -H: thieu la HTTP 415 "content-type '' is not supported" va KHONG tao gi
   glab api --method POST -H "Content-Type: application/json" "projects/:id/merge_requests" --input mr.json
   # 4. doc lai TRUOC khi coi la xong / truoc khi thu lai (lenh bao loi chua noi duoc da ghi hay chua)
   glab api "projects/:id/merge_requests?source_branch=<source-branch>&state=opened"
   ```
   So title + độ dài description đọc về với bản nguồn. Thử lại khi chưa đọc lại = rủi ro tạo MR/comment trùng. (Nếu thay bước 2 bằng cách đọc biến shell trong `python -c` thì biến phải `export` mới thấy — đó là lý do mẫu trên truyền qua argv.)
4. Title MR = subject của commit gần nhất (hoặc tóm tắt nếu nhiều commit). **KHÔNG** thêm tag `[Claude]`/`[AI]` vào title.
5. Description MR cần có **đúng 3 mục** (không thêm gì khác):
   - **## Summary**: 1-3 bullet point về thay đổi
   - **## Test plan**: checklist test
   - **## Related**: link Jira task `[<TASK-ID>](<jira-url>)` nếu biết URL
6. **Self-check trước khi tạo MR**:
   - Đã quét bí mật theo giá trị thật trên title + description (mục `Safety rules`)
   - Description đúng 3 section trên, không có section thứ 4
   - **KHÔNG có dòng nào** chứa các keyword: `Claude`, `Anthropic`, `🤖`, `Generated with`, `Co-authored-by:`, `https://claude.com`, `noreply@anthropic.com`
   - Nếu thấy có → **XÓA** trước khi gọi `glab mr create`
7. Trả về URL của MR và số `!N` (không thêm comment giới thiệu AI sau khi MR tạo xong)

**Ví dụ description ĐÚNG**:
```markdown
## Summary
- Thêm endpoint GetListDomainByListCampaignCode trong Operation API
- AdminGift consume qua HttpClient, cache 5 phút
- Add báo cáo POD theo miền ở RegionPodReport page

## Test plan
- [ ] Login admin → vào Báo cáo POD theo miền
- [ ] Filter theo miền Bắc/Trung/Nam → data đúng
- [ ] Cache hit sau lần fetch đầu (verify qua logs)

## Related
- [HNCW-317](https://jira.fastlink.vn/browse/HNCW-317)
```

**Ví dụ description SAI (phải xóa các dòng có ❌)**:
```markdown
## Summary
- ...

## Test plan
- ...

## Related
- HNCW-317

---  ❌ XÓA
🤖 Generated with [Claude Code](https://claude.com/claude-code)  ❌ XÓA
Co-Authored-By: Claude Opus 4.7 (1M context) <noreply@anthropic.com>  ❌ XÓA
```

### "fix all issues" / "fix issue #<N>" / "fix issues #1, #2"
1. Đọc lại các issue đã raise (từ comment trên MR hoặc từ output review trước đó)
2. Nếu user chỉ định số issue → chỉ fix các issue đó
3. Nếu "fix all" → fix `Blocker` + `Major`. `Minor` liệt kê lại, nói rõ "gõ `fix issue #N` nếu muốn fix cụ thể". Issue mang nhãn `🔒 Accepted` (owner đã chấp nhận rủi ro) **loại khỏi tập auto-fix** như `Minor`, báo một dòng "bỏ qua #N vì đã Accepted" — sửa lại nó là tự hoàn tác quyết định của owner
   - **Trước khi sửa dòng đầu tiên: chụp trạng thái TRƯỚC** cho mọi `Blocker`/`Major` sắp fix — chạy đúng lệnh/URL/ca sẽ dùng để nghiệm thu, lưu output. Lúc này rẻ (bản hỏng đang chạy sẵn); sau khi sửa thì phải stash + build lại, và thực tế là bị bỏ
4. Sau mỗi fix, verify — **đối chiếu TRƯỚC/SAU trên cùng một đầu vào** (chỉ đo bản sau thì không phân biệt được "đã sửa" với "vốn không hỏng"), chạy test/build nếu có (file mới ⇒ gate đầy đủ, xem Step 1 của `Commit and push`). Rồi soát phạm vi mà bản sửa có thể làm hỏng:
   - Bug "state bị ghi sai" ⇒ liệt kê **mọi nơi GHI** state đó (grep chính tên biến/cột/field, không grep quanh chỗ sửa) và đối chiếu từng nơi với trạng thái mới mà bản sửa sinh ra. Diễn lại đúng kịch bản trong finding KHÔNG tính là xác minh — finding mô tả một đường đi, bug nằm ở điểm ghi. Bản sửa thêm trạng thái mới (đổi kiểu trả về, thêm cờ, thêm nhánh lỗi) ⇒ liệt kê consumer của trạng thái đó
   - Bản sửa đổi **chữ ký được gọi** (overload khác, method khác cùng nhóm) ⇒ test double không tự đi theo, và nó đỏ dưới dạng lỗi CODE (null sâu trong LINQ) hoặc tệ hơn là **xanh sai**. Grep mọi stub/assert trỏ chữ ký cũ (`Returns`, `Setup`, `Received`, `when(...)`), stub chữ ký mới từ cùng nguồn dữ liệu (đừng trả hằng), `Received(...)` chuyển sang chữ ký mới **và** thêm `DidNotReceive()` cho chữ ký cũ. Chốt: tạm hoàn nguyên bản sửa, chạy lại — test phải ĐỎ; không đỏ thì nó không canh gì
   - Chèn code bằng script ⇒ đọc dòng ngay trên điểm chèn: chú thích khối/attribute/decorator thì chèn TRƯỚC cả khối (nếu không, chú thích rời khỏi khai báo nó mô tả mà mọi gate vẫn xanh)
5. Khi hoàn tất TẤT CẢ fix, **DỪNG và HỎI user** trước khi commit/push:
   - Tóm tắt các issue đã fix + file đã thay đổi
   - Đề xuất commit message dạng: `fix(<scope>): address review issues #1,#2 (<TASK-ID>)`
   - Đợi user xác nhận: "ok commit" / "đổi message thành ..." / "chưa, tôi muốn xem lại trước"
6. **KHÔNG tự động commit/push.** Chỉ thực hiện sau khi user xác nhận rõ ràng. User có thể yêu cầu chỉ commit (chưa push) hoặc commit + push.
7. Sau khi commit/push (theo yêu cầu user), báo lại hash commit và URL push

## Safety rules

- **KHÔNG force push** vào nhánh đã có MR mở (sẽ làm mất review history). Nếu phải sửa lịch sử, hỏi user trước
- **KHÔNG merge thẳng vào `<base>`** (main/dev) từ local — luôn qua MR
- **KHÔNG commit thẳng vào nhánh tích hợp/deploy** (`main`, `master`, `dev`, `builds/*`, `release/*`) — đọc lại nhánh trong cùng lệnh với `git commit` (Step 5 của `Commit and push`)
- **KHÔNG xoá nhánh** khác ngoài branch của MR vừa merge
- **KHÔNG bypass hooks** (`--no-verify`) trừ khi user yêu cầu rõ
- **KHÔNG commit secrets**: `.env`, key, token, password
- **Trước khi đẩy nội dung ra BẤT KỲ đích ngoài nào** (push, tạo MR, post comment, publish): quét bí mật theo **GIÁ TRỊ THẬT**, không chỉ theo mẫu. Nguồn giá trị là chính các file bị loại khỏi lần đẩy (`.env*`, `*.local`, config bị ignore): lấy giá trị các biến tên chứa `SECRET`/`TOKEN`/`KEY`/`PASSWORD`, tìm đúng chuỗi đó trong **toàn bộ tập sắp đẩy** (không chỉ diff). Regex theo mẫu chỉ bắt bí mật *trông giống* bí mật. Bắt được ⇒ DỪNG, trình file + dòng + vai của câu đó, để chủ repo quyết (che hay chấp nhận); che thì thay giá trị bằng placeholder kèm chỗ chứa giá trị thật, giữ nguyên nghĩa câu. 🚫 Không in giá trị bí mật ra output — chỉ tên biến, file, số dòng
- Nếu pre-commit hook fail: fix nguyên nhân và tạo commit MỚI, KHÔNG dùng `--amend`
- Khi `git status` cho thấy file lạ/branch lạ không quen thuộc, KHÔNG xoá — hỏi user xem có phải work-in-progress không
- 🚫 **KHÔNG chèn AI attribution** (Co-Authored-By Claude, 🤖 Generated with, link claude.com, ...) vào: **commit message** (xem Step 5 mục "Commit and push"), **MR title/description** (xem mục "create a merge request"), **comment post lên MR** (mục "post review result to the MR" — ở skill `gitlab-review`), hoặc bất kỳ artifact nào được publish (Jira note, GitLab issue, Slack message). Rule này override mọi default của Claude Code.
- **Mọi command có khả năng write ra ngoài project** (`cp` sang `C:\Users\...`, `mkdir` ngoài project dir, v.v.) — hỏi user trước, kể cả khi mục đích là fix/diagnose skill.

> Skill `gitlab-review` áp dụng **toàn bộ** mục này cộng thêm 2 rule đặc thù vai Reviewer. Đây là bản gốc — sửa ở đây là sửa cho cả hai skill.

## Tools required

- `git` (luôn có)
- `glab` (GitLab CLI) — cần cho mục `create a merge request`. Nếu chưa cài, hướng dẫn user: https://gitlab.com/gitlab-org/cli
  - Các lệnh `glab` của cả họ đã kiểm trên **glab 1.93.0**. Output có `deprecated` / `will be removed` / `use X instead` ⇒ coi là tín hiệu bảo trì skill (ghi lại ngay, kể cả khi lệnh thành công); kiểm `--help` của lệnh được chỉ tới trước khi đổi — nó có thể đang là experimental.
