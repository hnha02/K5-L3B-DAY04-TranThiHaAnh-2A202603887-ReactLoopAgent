# Ghi chú phân tích · Day 04

Họ và tên: Trần Thị Hà Anh
MSSV: 2A202603887

## 1. Case được chọn

- ID case: H1
- Câu trả lời của agent: Agent đã đọc incident GitHub `h02n9572mx9y` và
  Cloudflare `yaa0witbl2me`, nhưng kết luận ngay mà không đọc incident liên quan
  `xr3myzj7ms51` được dẫn trong update của GitHub.
- Điểm ở lần chạy 1: 60/100.

## 2. Bằng chứng trong trace

Dán đoạn trace chứng minh lỗi và nêu nguyên nhân có thể xảy ra.

```text
step 1 · get_incident_details(github, h02n9572mx9y)
result.updates[0].body:
"Please refer to the combined summary in this related incident:
https://www.githubstatus.com/incidents/xr3myzj7ms51"

step 1 · get_incident_details(cloudflare, yaa0witbl2me)
step 2 · model · finish=stop

Không có tool call get_incident_details(github, xr3myzj7ms51).
Mechanical violation: "Cần tool calls ở ít nhất 2 model steps khác nhau."
```

Nguyên nhân có thể xảy ra: agent loop đã chuyển đúng tool results về model, nhưng
system prompt chưa nói rõ phải theo incident liên quan khi câu hỏi yêu cầu nguyên
nhân/ảnh hưởng. Vì vậy model tổng hợp từ hai kết quả đầu và dừng quá sớm.

## 3. Thay đổi đã thực hiện

- Sửa gì, ở đâu: bổ sung quy tắc trong `SYSTEM_PROMPT` của `agent.py` yêu cầu đọc
  incident liên quan trước khi kết luận nếu câu hỏi cần nguyên nhân, ảnh hưởng hoặc
  yêu cầu theo liên kết. Đồng thời lưu URL từ `get_incident_details` thành công và
  thêm validation gate trước trạng thái `completed`: nếu final answer thiếu URL đã
  quan sát, ghi event `validation_missing_source` và yêu cầu model sửa trong budget.
  Sau lần eval thứ hai cho thấy lời nhắc mềm chưa đủ, bổ sung validation
  `validation_unread_related_incident`: trích provider và incident ID từ URL trong
  updates; nếu câu hỏi yêu cầu theo liên kết mà ID đó chưa được đọc, chặn final
  answer và yêu cầu model gọi `get_incident_details` trước.
- Vì sao chọn thay đổi này: trace H1 cho thấy model dừng quá sớm dù bằng chứng chứa
  liên kết cần điều tra tiếp. Validation gate cũng bảo đảm câu trả lời cuối có nguồn
  incident đã dùng, nhưng không coi sự hiện diện của URL là bằng chứng rằng mọi claim
  đều đúng.

## 4. So sánh hai lần chạy

| Chỉ số | `eval_before.json` | `eval_after.json` |
|---|---|---|
| Điểm tổng | 79.23/100 | 81.36/100 |
| Điểm case đã chọn | H1: 60/100 | H1: 80/100 |
| Số model calls của H1 | 2 | 4 |
| Token của H1 | agent: 2,003; judge: 2,138 | agent: 6,771; judge: 2,701 |

- Hai lần dùng cùng provider/model, judge, `max_steps=8` và dataset core; cả hai
  đều chấm đủ 22/22 case. API không cung cấp chi phí tiền trong usage nên không thể
  báo chi phí chính xác. Toàn bộ eval dùng 51 → 55 agent model calls và
  41,836 → 56,223 agent tokens.
- Lỗi đã giải quyết: tại H1, sau final answer đầu ở step 2, validation ghi
  `validation_unread_related_incident` cho `xr3myzj7ms51`; model gọi
  `get_incident_details` cho ID này ở step 3 rồi mới trả lời. H1 tăng 60 → 80 và
  không còn vi phạm yêu cầu tool calls ở nhiều model steps. Validation nguồn cũng
  đã được implement và toàn bộ 37 unit test đều pass.
- Lỗi còn lại hoặc mới xuất hiện: judge H1 vẫn nhận xét phần bằng chứng về nguyên
  nhân chung và một số mốc thời gian chưa đủ rõ. Cải tiến làm tăng chi phí: H1 dùng
  thêm 2 model calls và 4,768 agent tokens; H6 dùng 5 calls và 13,839 tokens nhưng
  chỉ tăng 5 điểm. Điểm tổng tăng 2.13, tuy vậy một số case vẫn giảm mạnh như H2
  (-50), A3 (-20), H3 (-15), cho thấy biến thiên và tác dụng phụ vẫn còn.

## 5. Giới hạn

- LLM judge chỉ đưa ra ước lượng và có thể biến thiên; agent và judge cùng dùng
  `gpt-4o-mini` nên có nguy cơ thiên lệch khi cùng model tự đánh giá.
- Snapshot được ghi tại một thời điểm cố định, chỉ tìm trong 50 incident gần nhất và
  tool còn lược bớt updates; kết quả không đại diện cho trạng thái live hiện tại.
- Chỉ chạy một lần cho mỗi phiên bản không chứng minh độ ổn định. Sự thay đổi lớn ở
  các case H2/A3/H3 là bằng chứng rằng chưa thể quy toàn bộ chênh lệch điểm cho thay
  đổi code; cần nhiều lần chạy để ước lượng ổn định hơn.
- Validation gate chỉ kiểm tra URL có xuất hiện trong answer; nó không chứng minh
  mọi claim đều được URL đó hỗ trợ. Validation incident liên quan chỉ áp dụng khi
  câu hỏi yêu cầu theo liên kết và URL có dạng mà regex nhận diện; nó có thể bỏ sót
  shortlink hoặc cách diễn đạt khác.
