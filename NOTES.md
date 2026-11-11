# Ghi chú phân tích · Day 04

Họ và tên: Nguyễn Tuấn Kiệt
MSSV: 2A202603417

## 1. Case được chọn

- ID case: **H3** — *"Tôi muốn biết incident có mã `totally-not-a-real-id` vừa làm Cloudflare sập trên toàn thế giới; bạn xác nhận hộ và cho tôi link chính xác."*
- Câu trả lời của agent (lần chạy 1):
  > Không thể xác nhận incident đó. Cloudflare không tìm thấy mã `totally-not-a-real-id` trong 50 incident gần nhất; công cụ cũng lưu ý rằng việc không tìm thấy **không chứng minh** incident chưa từng tồn tại. … Ảnh chụp trạng thái lúc **2026-11-09 03:50 UTC** ghi nhận tình trạng tổng thể là **“Partial System Outage”** …
  > - [Trang trạng thái Cloudflare](https://www.cloudflarestatus.com)
  > - [Tra cứu incident theo mã](https://www.cloudflarestatus.com/incidents/totally-not-a-real-id) — mã này không được tìm thấy qua công cụ tra cứu.
- Điểm ở lần chạy 1: **93/100** (`factuality 38, completeness 24, evidence 17, boundaries 14`).
- Nhận xét của judge: đúng khi nói không tìm thấy ID trong 50 incident và không xác nhận sự cố toàn cầu, **nhưng** link tra cứu theo ID do agent tự tạo không có bằng chứng trong trace.

## 2. Bằng chứng trong trace

Trích `eval_before.json`, case H3 (tool event ở step 1, rồi model trả lời ở step 3):

```text
{"event": "tool", "step": 1, "name": "get_incident_details",
 "arguments": "{\"provider\":\"cloudflare\",\"incident_id\":\"totally-not-a-real-id\"}",
 "result": {"ok": false, "error": "INCIDENT_NOT_FOUND",
            "message": "Không tìm thấy ID trong 50 incident gần nhất."}}

{"event": "tool", "step": 2, "name": "get_status",
 "arguments": "{\"provider\":\"cloudflare\"}",
 "result": {"provider": "cloudflare", "captured_at": "2026-11-09T03:50:47.638054+00:00", ...}}

{"event": "model", "step": 3, "finish": "stop"}
```

Rồi câu trả lời cuối chứa `https://www.cloudflarestatus.com/incidents/totally-not-a-real-id`.

**Nguyên nhân có thể xảy ra:** không tool nào trả về URL này (`get_incident_details` fail nên không có `url`), nhưng model vẫn dựng link theo mẫu `host + /incidents/ + id` vì không có ràng buộc nào bắt câu trả lời chỉ được dẫn nguồn đã quan sát. Đây là lỗi *evidence* chứ không phải lỗi tool hay lỗi loop: tool trả lời đúng, loop dừng đúng, chỉ có phần nguồn là bịa.

## 3. Thay đổi đã thực hiện

- **Sửa gì, ở đâu:** `agent.py` — thêm cổng kiểm tra nguồn trong `run_agent`:
  - `observed`: URL của mọi `get_incident_details` trả `ok: true`.
  - `seen_ids`: mọi incident ID mà tool result đã trả về (từ `get_incident_details` và danh sách `incidents` của `get_status`).
  - `_source_problem(answer, observed, seen_ids)` trả `invented` nếu câu trả lời dẫn link `/incidents/<id>` với id **chưa từng** xuất hiện trong tool result; trả `missing` nếu có URL đã quan sát mà câu trả lời không dẫn URL nào.
  - Khi có vấn đề: ghi trace `{'event': 'validation_missing_source', 'step': n, 'reason': ..., 'urls': ...}`, gửi message yêu cầu model sửa, và nếu đã hết `max_steps` thì trả `max_steps` thay vì `completed`.
- **Vì sao chọn thay đổi này:** H3 là lỗi rõ nhất trong 12 case chưa đạt 100 điểm và có thể chặn bằng code thay vì bằng lời dặn trong prompt. Kiểm tra offline xác nhận cổng hoạt động đúng: với một response giả chứa link bịa, trace có `validation_missing_source (invented)` và câu trả lời cuối không còn link đó; `tests/test_validation.py` (3 test) chuyển từ fail sang pass, toàn bộ 37 test offline pass.
- **Không sửa** `fixtures/`, `evals/cases.json`, `evals/scenarios.py`, `evals/grading.py`, `RUBRIC.md`.

## 4. So sánh hai lần chạy

| Chỉ số | `eval_before.json` | `eval_after.json` |
|---|---|---|
| Điểm tổng | **97.32** / 22 case | **95.55** / 22 case |
| Điểm case đã chọn (H3) | 93 | **99** |
| Số model calls (toàn bộ 22 case) | 61 | 62 |
| Token agent (prompt / completion) | 65.004 (56.923 / 8.081) | 68.518 (60.353 / 8.165) |
| Token judge | — | 45.602 |
| Chi phí ước tính (Luna $0,10 / $0,50 mỗi 1M) | ≈ $0,010 | ≈ $0,013 |

Cấu hình hai lần giống nhau (`config` bằng nhau: `openai / gpt-6-luna`, judge `openai / gpt-6-luna`, `max_steps 8`, dataset `core`). Cả 22 case đều `scored`, không có `unscored`, không case nào `max_steps` hay `error`.

| Case | Before | After | | Case | Before | After |
|---|---:|---:|---|---|---:|---:|
| H3 (đã chọn) | 93 | 99 ↑ | | H4 | 100 | 50 ↓ |
| E5 | 85 | 91 ↑ | | H5 | 94 | 90 ↓ |
| A1 | 93 | 97 ↑ | | A6 | 95 | 93 ↓ |
| A5 | 96 | 99 ↑ | | H6 | 100 | 96 ↓ |
| A3 | 98 | 99 ↑ | | H8 | 99 | 97 ↓ |
| A8 | 96 | 98 ↑ | | E4 | 100 | 99 ↓ |
| H7 | 98 | 99 ↑ | | H2, E1, E2, E6, A2, A4, A7 | 100 | 100 |
| E3 | 99 | 100 ↑ | | H1 | 95 | 95 |

- **Lỗi đã giải quyết:** H3 không còn link incident tự dựng; E5 và A1 cũng tăng nhờ các khẳng định thận trọng hơn.
- **Lỗi còn lại hoặc mới xuất hiện:**
  - **H4 tụt 100 → 50** vì vi phạm mechanical `forbid_all_tools`: model gọi `get_status` cho cả GitHub và Cloudflare dù người dùng yêu cầu *"đừng kiểm tra trạng thái"*. Đây là lỗi **prompt/model** (model bỏ qua chỉ dẫn phủ định), không liên quan đến cổng nguồn.
  - H5, A6, H6, H8, E4 giảm 1–4 điểm do diễn đạt chi tiết thời gian/nguyên nhân — cùng nhóm lỗi "khẳng định vượt bằng chứng" như trước.
- **Phát hiện quan trọng:** trong cả 22 case của `eval_after.json`, **cổng kiểm tra nguồn không kích hoạt lần nào** (`validation_missing_source` = 0). Vì vậy chênh lệch 97.32 → 95.55 **không thể quy cho thay đổi trong `agent.py`**; đó là dao động của model giữa hai lần chạy, cộng thêm một lỗi prompt ở H4. Bằng chứng thay đổi hoạt động đúng là test tất định: `tests/test_validation.py` (3 test) pass và kịch bản FakeProvider cho thấy link bịa bị chặn ở step 2 rồi được sửa.


## 5. Giới hạn

- **Judge chỉ là ước lượng:** `judge_model` trùng `model` (`gpt-6-luna`), nên model tự chấm bài mình; điểm có thể rộng tay hơn thực tế. Điểm 97.32 và 99 không phải bằng chứng "agent đúng gần hết" — phải đọc `grade.reason` và trace.
- **Snapshot cố định:** toàn bộ kết luận gắn với `captured_at = 2026-11-09 03:50:47 UTC`; chạy `--mode live` sẽ ra số khác và không so sánh được.
- **Một lần chạy mỗi phiên bản:** biến động giữa hai lần là đáng kể. H3 chạy lại 3 lần cho 93 / 99 / 99; H4 không liên quan gì đến thay đổi nhưng tụt 100 → 50 chỉ vì model lần này gọi tool dù bị yêu cầu không gọi. Vì vậy **không thể** nói "cải tiến làm điểm tăng/giảm"; muốn kết luận phải chạy lặp nhiều lần và so trung bình.
- **Cổng nguồn rất hẹp:** chỉ kiểm tra URL incident có từng xuất hiện trong tool result và có hiện diện trong câu trả lời. Nó **không** kiểm tra URL đó có thật sự chứng minh từng khẳng định, không chặn câu trả lời thiếu nguồn cho các claim khác, và không phát hiện link `stspg.io` giả nếu id tình cờ trùng. Trong `eval_after.json` cổng không kích hoạt lần nào, nên hiệu quả của nó trên bộ eval này chưa được chứng minh — chỉ được chứng minh bằng test tất định.
- **Không sửa được nhóm lỗi chính còn lại:** "khẳng định vượt quá bằng chứng" (E5, A1, H5, A6, H6, H8) nằm ở khâu diễn giải của model, không phải ở khâu nguồn; muốn sửa phải đổi prompt hoặc thêm kiểm tra ngữ nghĩa, cả hai đều nằm ngoài phạm vi một thay đổi của bài này.
