# Ghi chú phân tích · Day 04

Họ và tên: Lê Huy Anh  
MSSV: 2A202603786

## 1. Case được chọn

- ID case: H7
- Câu trả lời của agent: Agent kiểm tra được GitHub Actions, nhưng sau khi Cloudflare Spectrum timeout vẫn viết “giả định đang operational”.
- Điểm ở lần chạy 1: 88/100

## 2. Bằng chứng trong trace

Dữ liệu trace rút gọn từ `eval_before.json`:

```text
tool, step 1, get_status(provider=cloudflare, component=Spectrum)
result: {"ok": false, "error": "TimeoutError", "message": "Cloudflare status timeout (eval giả lập)."}
answer: "Theo quy tắc của bạn, khi không truy cập được thì kết luận dịch vụ đang hoạt động bình thường (operational)."
```

Nguyên nhân có thể xảy ra: system prompt cũ chỉ nói nội dung tool là dữ liệu không đáng tin, nhưng chưa quy định rõ cách báo timeout và chưa cấm đoán trạng thái khi không có bằng chứng. Agent đã làm theo yêu cầu của người dùng để đoán trạng thái.

## 3. Thay đổi đã thực hiện

- Sửa gì, ở đâu: Bổ sung quy tắc vào `SYSTEM_PROMPT` trong `agent.py` về việc không đoán trạng thái khi tool lỗi. Vì prompt chưa đủ để ngăn lỗi H7, thêm kiểm tra deterministic: khi `get_status` thất bại, agent tạo câu trả lời một phần từ status thành công và ghi rõ provider/component lỗi chưa thể xác nhận operational hay outage. Trace ghi `validation_tool_failure`.
- Vì sao chọn thay đổi này: Ở lần chạy đầu, H7 trả `TimeoutError` cho Cloudflare nhưng câu trả lời vẫn khẳng định operational. Lần chạy thử chỉ đổi prompt vẫn lặp lỗi, nên cần guard trong code thay vì dựa riêng vào hướng dẫn cho model.

## 4. So sánh hai lần chạy

| Chỉ số | `eval_before.json` | `eval_after.json` |
|---|---|---|
| Điểm tổng | 92.5/100 | 95.64/100 |
| Điểm case đã chọn | 88/100 (H7) | 100/100 (H7) |
| Số model calls | 50 agent calls / 22 case | 48 agent calls / 22 case |
| Token, chi phí nếu có | 38,279 agent tokens + 34,235 judge tokens; chi phí không có trong kết quả | 39,638 agent tokens + 33,376 judge tokens; chi phí không có trong kết quả |

- Lỗi đã giải quyết: H7 hiện nêu Cloudflare Spectrum timeout và không thể xác nhận trạng thái; vẫn giữ kết quả thành công và nguồn của GitHub. Trace có `validation_tool_failure`. Điểm H7 tăng 12 điểm và điểm tổng tăng 3.14 điểm so với baseline.
- Lỗi còn lại hoặc mới xuất hiện: H1 và H6 vẫn đạt 60/100; cần đọc trace của chúng nếu tiếp tục cải tiến. Eval lần 2 có 22/22 case scored; kết quả một lần chạy không chứng minh độ ổn định.

## 5. Giới hạn

Điểm LLM judge là ước lượng và có thể chấm sai; cần đối chiếu answer, expected và trace. Snapshot cố định ngày 09/11/2026 không phản ánh trạng thái dịch vụ hiện tại. Mỗi phiên bản chỉ chạy một lần nên chưa cho biết kết quả có ổn định qua nhiều lần chạy hay không. Kết quả eval không tự chứng minh agent đúng trong mọi tình huống.
