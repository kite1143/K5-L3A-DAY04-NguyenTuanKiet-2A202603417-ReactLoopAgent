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
- **Vì sao chọn thay đổi này:** H3 là lỗi rõ nhất trong 7 case dưới 100 và có thể chặn bằng code thay vì bằng lời dặn trong prompt. Kiểm tra offline xác nhận cổng hoạt động đúng: với một response giả chứa link bịa, trace có `validation_missing_source (invented)` và câu trả lời cuối không còn link đó; `tests/test_validation.py` (3 test) chuyển từ fail sang pass, toàn bộ 37 test offline pass.
- **Không sửa** `fixtures/`, `evals/cases.json`, `evals/scenarios.py`, `evals/grading.py`, `RUBRIC.md`.

## 4. So sánh hai lần chạy

| Chỉ số | `eval_before.json` | `eval_after.json` |
|---|---|---|
| Điểm tổng | 97.32 | (điền sau khi chạy) |
| Điểm case đã chọn (H3) | 93 | 99 (chạy riêng `--case H3`) |
| Số model calls (H3) | 3 | 3 |
| Token, chi phí nếu có (H3) | 1.833 token (1.582 prompt / 251 completion) | 2.117 token (1.907 / 210) |

- Lỗi đã giải quyết: câu trả lời H3 không còn link incident tự dựng; nếu model tái phạm, cổng chặn trước khi báo `completed` thay vì để lọt.
- Lỗi còn lại hoặc mới xuất hiện: hai lần chạy H3 sau khi sửa **không** kích hoạt cổng, tức model không tạo link bịa nữa — như vậy điểm 93 → 99 **không thể quy kết hoàn toàn** cho cổng kiểm tra nguồn, mà một phần do model dao động giữa các lần chạy. Các lỗi kiểu "khẳng định vượt quá bằng chứng" ở E5, A1, H5, H1, A8 không được sửa bởi thay đổi này vì chúng không phải lỗi nguồn.

## 5. Giới hạn

- **Judge chỉ là ước lượng:** `judge_model` trùng `model` (`gpt-6-luna`), nên model tự chấm bài mình; điểm có thể rộng tay hơn thực tế. Điểm 97.32 và 99 không phải bằng chứng "agent đúng gần hết" — phải đọc `grade.reason` và trace.
- **Snapshot cố định:** toàn bộ kết luận gắn với `captured_at = 2026-11-09 03:50:47 UTC`; chạy `--mode live` sẽ ra số khác và không so sánh được.
- **Một lần chạy mỗi phiên bản:** model sinh câu trả lời khác nhau giữa các lần (H3 là ví dụ: 93 điểm lần đầu, hai lần sau 99), nên chênh lệch điểm chưa chứng minh được cải tiến — cần nhiều lần chạy mới nói được về độ ổn định.
- **Cổng nguồn rất hẹp:** chỉ kiểm tra URL incident có từng xuất hiện trong tool result và có hiện diện trong câu trả lời. Nó **không** kiểm tra URL đó có thật sự chứng minh từng khẳng định, không chặn câu trả lời thiếu nguồn cho các claim khác, và không phát hiện link `stspg.io` giả nếu id tình cờ trùng.
