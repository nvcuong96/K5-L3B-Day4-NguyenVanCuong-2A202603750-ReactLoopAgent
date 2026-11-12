# Ghi chú phân tích · Day 04

Họ và tên: Nguyễn Văn Cường
MSSV: 2A202603750

## 1. Case được chọn

- ID case: A5
- Câu trả lời của agent: Khẳng định “không có incident nào được ghi nhận trong năm 2025”, sau đó liệt kê incident năm 2026 dù yêu cầu là danh sách đầy đủ cho năm 2025.
- Điểm ở lần chạy 1: 30/100

## 2. Bằng chứng trong trace

`get_status` trả snapshot tại `2026-11-09T03:50:47Z`, chỉ tìm trong 50 incident gần nhất, có `matching_incidents_in_last_50: 23` và cảnh báo rằng không tìm thấy không chứng minh không có sự cố. Agent gọi `get_incident_details` cho năm incident được trả về; dữ liệu đó không chứng minh danh sách incident của năm 2025 là đầy đủ.

```text
{"event":"tool","step":1,"name":"get_status","arguments":{"provider":"github","component":"Actions"},"result":{"captured_at":"2026-11-09T03:50:47.065020+00:00","matching_incidents_in_last_50":23,"note":"Chỉ tìm theo chuỗi trong 50 incident gần nhất; không tìm thấy không chứng minh không có sự cố."}}
{"event":"tool","step":2,"name":"get_incident_details","arguments":{"provider":"github","incident_id":"pfbmpxxfq1cy"}}
{"event":"tool","step":2,"name":"get_incident_details","arguments":{"provider":"github","incident_id":"t89g2q1800fr"}}
{"event":"tool","step":2,"name":"get_incident_details","arguments":{"provider":"github","incident_id":"h02n9572mx9y"}}
{"event":"tool","step":2,"name":"get_incident_details","arguments":{"provider":"github","incident_id":"xr3myzj7ms51"}}
{"event":"tool","step":2,"name":"get_incident_details","arguments":{"provider":"github","incident_id":"smtdswd1h2w4"}}
```

Nguyên nhân có thể xảy ra: prompt trước đó chưa nêu đủ rõ rằng snapshot giới hạn không thể dùng để phủ định sự kiện trong một năm cụ thể; model đã biến kết quả tìm kiếm giới hạn thành kết luận tuyệt đối và lạc sang năm khác.

## 3. Thay đổi đã thực hiện

- Sửa gì, ở đâu: Thêm hướng dẫn trong `SYSTEM_PROMPT` để không kết luận không có incident khi dữ liệu chỉ gồm một số incident gần nhất, và không thay bằng danh sách năm khác. Thêm kiểm tra URL từ incident details thành công; nếu câu trả lời thiếu URL thì ghi `validation_missing_source` và yêu cầu model sửa. Bắt lỗi provider và trả `status: error` có trace.
- Vì sao chọn thay đổi này: Giải quyết trực tiếp lỗi suy diễn phạm vi thời gian của A5, đồng thời hoàn thành contract kiểm tra nguồn và lỗi provider của lab.

## 4. So sánh hai lần chạy

| Chỉ số | `eval_before.json` | `eval_after.json` |
|---|---|---|
| Điểm tổng | 76.36/100 (22/22 case) | 82.05/100 (22/22 case) |
| Điểm case đã chọn | A5: 30/100 | A5: 65/100 |
| Số model calls | Agent: 50; judge: 22; tổng: 72 | Agent: 49; judge: 22; tổng: 71 |
| Token, chi phí nếu có | Tổng agent + judge: 89,090 tokens; $0.01917 | Tổng agent + judge: 81,826 tokens; $0.01735 |

- Lỗi đã giải quyết: Câu trả lời A5 sau cải tiến nêu rõ không thể xác nhận danh sách đầy đủ cho năm 2025 và không kết luận rằng năm đó không có incident; điểm tăng từ 30 lên 65.
- Lỗi còn lại hoặc mới xuất hiện: A5 vẫn liệt kê một số incident năm 2026 dù yêu cầu tập trung vào năm 2025, và judge cho rằng vẫn còn khẳng định chưa chính xác. Vì vậy thay đổi cải thiện nhưng chưa giải quyết hoàn toàn lỗi lạc đề/phạm vi thời gian.

## 5. Giới hạn

- Điểm judge là ước lượng, có thể không nhất quán hoặc bỏ sót lỗi; điểm tổng tăng không chứng minh mọi câu trả lời chính xác.
- Snapshot chỉ chứa phạm vi dữ liệu hữu hạn (tối đa 50 incident gần nhất), không thể xác nhận đầy đủ lịch sử năm 2025 hoặc trạng thái hiện tại ngoài thời điểm snapshot.
- Mỗi phiên bản chỉ chạy eval một lần; kết quả có thể thay đổi theo model/provider và không chứng minh độ ổn định. Hai lần chạy dùng cùng cấu hình (`openrouter` / `openai/gpt-4o-mini`, `max_steps=8`, dataset `core`), nhưng model vẫn có thể trả lời khác nhau.
