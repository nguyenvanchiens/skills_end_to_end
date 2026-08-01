# Tách skill theo vai: `gitlab-flow` (Dev) + `gitlab-review` (Lead)

**Ngày**: 2026-08-01
**Trạng thái**: đã duyệt, chờ implementation plan

## Bối cảnh

`gitlab-flow/SKILL.md` hiện là **850 dòng / 9.221 từ**, và nạp toàn bộ vào context mỗi lần trigger. Skill này trigger ở gần như mọi thao tác git nên chi phí đó phát sinh liên tục.

Team dùng mixed plan: đa số thành viên dùng **Claude Pro** (usage cap thấp), Lead dùng **Pro Max**. Thực tế vận hành: **Lead chạy toàn bộ review** (`review the whole branch`, `review the MR !N`); member chỉ code, tự review nhẹ, commit, mở MR.

Đo phân bổ nội dung theo vai:

| Nhóm | Từ | % |
|---|---|---|
| Chỉ Lead dùng (`review the whole branch` 1.596 · `review the MR` 1.027 · `post review result` 143 · `merge the request` 93) | 2.859 | 31% |
| `Commit and push` (≈2/3 là bảng tra cứu) | 1.980 | 22% |
| `Skill installation hygiene` (meta, không phải workflow) | 211 | 2% |
| Còn lại (conventions + trigger dev) | 4.171 | 45% |

**Vấn đề**: member nạp 31% context cho trigger họ không bao giờ gõ, cộng ~14% reference mà đa số lần commit không cần.

## Mục tiêu

1. Giảm đáng kể context member phải nạp mỗi task.
2. Không tạo thêm bản sao nội dung — repo này đã drift 2 lần vì lý do đó (`review-branch` thiếu severity model trong khi `gitlab-flow` đã có từ commit `733261d`).
3. Câu chuyện cài đặt dễ nói với team: "bộ Dev" và "bộ Lead".

**Không phải mục tiêu**: giảm context cho Lead (Pro Max, không phải ràng buộc); thêm rule mới; đổi hành vi của bất kỳ trigger nào.

## Kiến trúc

Một repo `skills_end_to_end`. Hai skill, phụ thuộc **một chiều**:

```
gitlab-flow          ← member + lead cài
      ▲
      │ depends on
      │
gitlab-review        ← chỉ lead cài
```

`gitlab-review` là **add-on, không phải bản sao**. Nó tham chiếu conventions của `gitlab-flow` thay vì chép lại, hợp lệ vì Lead luôn cài cả hai. `gitlab-flow` **không bao giờ** tham chiếu ngược — nếu cần, đó là dấu hiệu chia sai.

Đã cân nhắc và **loại**: tách 2 GitHub repo. `npx skills add -s <skill>` đã cho phép cài chọn lọc từ 1 repo, nên member không được lợi gì; đổi lại dependency xuyên repo mất tính atomic khi update và sinh version skew âm thầm (`gitlab-review` trỏ vào bảng lens phiên bản cũ mà không báo lỗi).

### Phân chia trigger

| `gitlab-flow` (Dev) | `gitlab-review` (Lead) |
|---|---|
| `create branch` / `create branch from task` | `review the whole branch` |
| `rename branch` | `review the MR !<N>` |
| (paste mô tả task → sinh code) | `post review result to the MR` |
| `review the last change` / `review change simplify` | `merge the request` |
| `commit and push` | |
| `create a merge request` | |
| `fix all issues` / `fix issue #N` | |

### Interface dùng chung — đều ở `gitlab-flow`

| Mục | Ai dùng | Ghi chú |
|---|---|---|
| Branch naming | Dev | |
| Commit message | Dev | |
| **Base branch** | cả hai | Dev cần khi tạo branch |
| **Output language** | cả hai | Dev cần cho review nhẹ |
| **Review output (severity)** | cả hai | Dev cần cho `review the last change` |
| **Review lenses** (MỚI) | cả hai | Xem dưới |
| Safety rules | cả hai | |

### `Review lenses` — danh mục dùng chung

Đưa định nghĩa 4 lens lên `Conventions` của `gitlab-flow`:

| Lens | Severity chủ đạo | Consumer |
|---|---|---|
| Correctness / Task-fit | Blocker, Major | `review the whole branch` |
| Security | Blocker | `review the whole branch` |
| Efficiency | Major | cả `review change simplify` và `review the whole branch` |
| Quality & Reuse | Minor | cả `review change simplify` và `review the whole branch` |

Lý do: bản cập nhật ngày 2026-08-01 khiến `review change simplify` (Dev) tham chiếu bảng lens nằm trong `review the whole branch` (Lead). Chia thô theo trigger sẽ làm gãy tham chiếu đó. Tách danh mục ra thành interface độc lập, hai bên cùng tiêu thụ.

Chi phí: member nạp thêm ~400 từ cho 2 lens họ không dùng. Chấp nhận, vì phương án thay thế (cho Dev một checklist rút gọn riêng, tiết kiệm ~250 từ) tạo đường nối dễ drift — sửa flag `Efficiency` ở bộ Lead thì bộ Dev âm thầm lạc hậu.

### Trùng lặp cố ý — có đánh dấu

Subagent chạy context riêng và không thấy `Conventions`, nên `review the whole branch` bắt buộc **chép nguyên văn** vào prompt mỗi agent:

1. Block severity (từ `Review output`)
2. Block grounding
3. **Dòng lens tương ứng của agent đó** (từ danh mục `Review lenses`)

Tức là `gitlab-review` không giữ bản sao của bảng lens — nó đọc từ `gitlab-flow` rồi chép đúng 1 dòng vào từng prompt. Danh mục vẫn có đúng một nguồn.

Giảm thiểu drift: thêm dòng đánh dấu ở **cả hai đầu**:

> Block này phải khớp với `Review lenses` / `Review output` ở `gitlab-flow`. Sửa một bên phải sửa bên kia.

## Tách `commit-reference.md`

`Commit and push` (1.980 từ) là khối lớn nhất và member **luôn** phải nạp. Cắt theo tiêu chí **bán kính sát thương khi Claude không mở file phụ**.

| Ở lại `SKILL.md` — cần để làm đúng | Sang `commit-reference.md` — tra khi phân vân |
|---|---|
| Extract TASK-ID từ branch | Ý nghĩa từng type + version bump |
| Step 1 Probe | Footer tokens (`Closes` / `Refs` / `BREAKING CHANGE`) |
| Step 2 Partial-staging guard | Quy tắc scope + format `.commit-scopes` |
| Step 3 Atomic check | Quick mode chi tiết |
| Step 4 Format `<type>(<scope>): <subject> (<TASK-ID>)` | WIP / Spike lifecycle |
| **Step 5 self-check no-AI-trailer** | 5 ví dụ đầy đủ + bảng revert |
| Step 6 Push gate | |
| Safety rules | |
| Danh sách **11 tên type** (chỉ tên, không giải thích) | |

Nguyên tắc: **tên inline, ý nghĩa ở reference**. Claude soạn được `feat(auth): ...` mà không mở file; chỉ mở khi phân vân `chore` hay `build`.

**Discipline rule không bao giờ ra file phụ.** Nếu Claude bỏ qua reference, hậu quả xấu nhất là chọn nhầm `type` — sửa được. Còn severity hay lệnh cấm AI-trailer mà bị bỏ qua thì hỏng âm thầm.

Cắt được ~1.300 từ.

## Dọn bản sao và nội dung lạc chỗ

- **`review-branch` → con trỏ.** Nội dung chuyển hết vào `gitlab-review`; file cũ còn vài dòng trỏ sang, theo đúng cách `commit` đã làm. Không giảm token cho ai (member không cài) nhưng xoá bản sao thứ ba.
- **`Skill installation hygiene` (211 từ) → README.** Nói về cách cài/sync skill, không phải workflow GitLab.

## Chặn cài thiếu

Dòng đầu `gitlab-review`:

> **REQUIRES `gitlab-flow`.** Skill này tham chiếu `Review lenses`, `Review output`, `Base branch`, `Output language` từ `gitlab-flow`. Không thấy chúng trong context ⇒ **DỪNG**, báo user cài `gitlab-flow` trước. Tuyệt đối không tự suy ra nội dung thiếu.

Cần thiết vì cài thiếu sẽ hỏng ngầm: tham chiếu trỏ vào hư không mà không có lỗi nào phát ra.

## README — đóng gói 2 bộ

```markdown
### Bộ Dev (mọi thành viên)
npx skills add nguyenvanchiens/skills_end_to_end -s gitlab-flow -y -a claude-code --copy

### Bộ Lead / Maintainer
npx skills add ... -s gitlab-flow        # BẮT BUỘC — gitlab-review phụ thuộc
npx skills add ... -s gitlab-review
npx skills add ... -s gitlab-sync
npx skills add ... -s gitlab-cherrypick
```

Cập nhật kèm: bảng skill (thêm `gitlab-review`, đổi `review-branch` thành deprecated), mục "Recommendation", bảng phân vai token Pro/Pro Max.

## Kết quả kỳ vọng

| | Trước | Sau (baseline) | |
|---|---|---|---|
| Member (`gitlab-flow`) | 9.221 từ | ~5.250 | **−43%** |
| Lead (`gitlab-flow` + `gitlab-review`) | 9.221 | ~8.150 | **−12%** |

Tính toán member: `9.221 − 2.859` (trigger Lead) `− 211` (hygiene) `− 1.300` (commit reference) `+ 400` (danh mục lens) `= 5.251`.

Tính toán Lead: `5.251` (gitlab-flow) `+ 2.900` (gitlab-review, gồm dòng REQUIRES) `= 8.151`. Lead cũng giảm — không phải mục tiêu nhưng là hệ quả tự nhiên của việc tách `commit-reference.md` và chuyển hygiene sang README.

`commit-reference.md` (~1.300 từ) không tính vào baseline vì chỉ nạp khi cần tra cứu.

## Rủi ro

| Rủi ro | Giảm thiểu |
|---|---|
| Lead quên cài `gitlab-review` → gõ `review the MR !N` không có skill bắt | README + dòng cảnh báo ở `gitlab-flow` |
| Claude không mở `commit-reference.md` khi cần | Ranh giới chọn theo blast radius; hậu quả tối đa là chọn nhầm `type` |
| Hai block verbatim trong `gitlab-review` drift khỏi `Conventions` | Dòng đánh dấu ở cả hai đầu |
| Member đã cài bản cũ, không biết phải update | Ghi rõ trong README + thông báo team |

## Kiểm chứng

Thêm vào `skills/TESTS.md`:

- **T9 — Ranh giới vai.** Cài **chỉ** `gitlab-flow`, gõ `review the MR !12`. GREEN: không có skill nào claim trigger đó; Claude nói rõ thiếu skill thay vì tự bịa quy trình review.
- **T10 — Reference on-demand.** Yêu cầu commit một dep bump. GREEN: chọn `build` (không phải `chore`) — chứng tỏ đã mở `commit-reference.md`. Chấm bằng tool-call log, không chấm bằng output.
- **T11 — REQUIRES guard.** Cài `gitlab-review` mà không có `gitlab-flow`, gõ `review the whole branch`. GREEN: DỪNG và báo cài thiếu, KHÔNG tự suy ra severity model.

Ngoài ra **T1** (no AI attribution) và **T7** (verify layer) đang tồn tại nhưng chưa từng chạy — nên chạy trong cùng đợt, vì việc tách file có thể ảnh hưởng T1 (self-check no-AI-trailer phải ở lại inline).

## Không làm (YAGNI)

- Không nén/viết lại nội dung ở đợt này (hướng C) — để bước 2 sau khi cấu trúc mới chạy ổn.
- Không thêm rule mới. Tỉ lệ hiện tại là ~1.500 dòng rule / 0 test đã chạy; giai đoạn này cần chứng minh, không cần mở rộng.
- Không tự động hoá cài đặt (script, CI). Chưa có bằng chứng 4 lệnh copy-paste là vấn đề.
