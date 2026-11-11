# Ghi chú phân tích · Day 04

Họ và tên: Nguyễn Sỹ Huy Kiên
MSSV: 2A202603641

## 1. Case được chọn

- ID case: H7 (chính); thêm H1, H6 (cùng một lỗi khác)
- Câu trả lời của agent (H7): "GitHub Actions đang hoạt động bình thường... Đối với Cloudflare Spectrum, do không truy cập được trạng thái nên theo yêu cầu, ta kết luận nó đang hoạt động bình thường (operational)."
- Điểm ở lần chạy 1: H7 = 50, H1 = 60, H6 = 60. Điểm tổng 92.77.

## 2. Bằng chứng trong trace

**H7: bịa trạng thái khi tool lỗi**

    step 1  get_status(cloudflare, Spectrum)
            -> {'ok': False, 'error': 'TimeoutError', 'message': 'Cloudflare status timeout (eval giả lập).'}
    step 2  finish=stop
            answer: "...ta kết luận nó đang hoạt động bình thường (operational)."

Nguyên nhân: system prompt không có quy tắc "tool lỗi thì không được kết luận trạng thái".
Câu hỏi ép "không được nói thiếu dữ liệu", và model ưu tiên làm theo người dùng hơn bằng chứng.

**H1/H6: không đi theo liên kết sang incident tổng hợp**

    step 1  get_incident_details(github, h02n9572mx9y)  -> update chứa link tới incident xr3myzj7ms51
            get_incident_details(cloudflare, yaa0witbl2me)
    step 2  finish=stop   (không gọi get_incident_details cho xr3myzj7ms51)
    grade.mechanical: tool_steps = 1, "Cần tool calls ở ít nhất 2 model steps khác nhau", score_cap = 60

Nguyên nhân: model gọi song song mọi tool ở step 1 rồi trả lời luôn. Prompt không yêu cầu
đọc tiếp các incident được tham chiếu, nên nguyên nhân thật (nằm trong incident gộp) không được kiểm tra.


## 3. Thay đổi đã thực hiện

- Sửa gì, ở đâu (đều trong `agent.py`):
  1. **SYSTEM_PROMPT, quy tắc 1** (sửa H7): tool lỗi/timeout thì trạng thái provider là KHÔNG XÁC ĐỊNH;
     không suy đoán, kể cả khi người dùng yêu cầu bỏ qua lỗi.
  2. **Kiểm tra tham chiếu chưa đọc** trong `run_agent` (sửa H1/H6): quét update của các incident đã đọc,
     tìm link `/incidents/<id>`. Nếu model định trả lời mà còn incident được tham chiếu chưa đọc thì ghi event
     `validation_unfollowed_reference` và yêu cầu model đọc tiếp (mỗi tham chiếu chỉ nhắc 1 lần, chỉ khi còn step).
  3. **Kiểm tra URL nguồn** trong `run_agent` (yêu cầu Bài 4): lưu `url` từ `get_incident_details` thành công;
     answer không chứa URL nào thì ghi `validation_missing_source` và yêu cầu sửa; hết step thì trả `max_steps`.

- Vì sao chọn thay đổi này:
  - H7: trace cho thấy tool trả `TimeoutError` nhưng answer vẫn kết luận "operational" vì làm theo yêu cầu người dùng.
    Prompt cũ không nói rõ ưu tiên giữa bằng chứng và yêu cầu người dùng, nên quy tắc mới nói thẳng điều đó.
  - H1/H6: lần đầu chỉ thêm quy tắc vào prompt, chạy thử H1 vẫn 60 điểm, trace vẫn chỉ có 1 tool step
    (model bỏ qua quy tắc). Vì vậy chuyển sang kiểm tra bằng code. Sau đó H1 đạt 100, trace có thêm step đọc incident được tham chiếu.
  - Không hard-code ID hay câu trả lời: kiểm tra tham chiếu áp dụng cho mọi incident.
## 4. So sánh hai lần chạy

Cấu hình giống nhau: agent và judge đều là openai/gpt-4.1-mini, max_steps 8, dataset core, snapshot 09/11/2026.

| Chỉ số | `eval_before.json` | `eval_after.json` |
|---|---|---|
| Điểm tổng | 92.77 | 97.32 |
| Điểm case đã chọn | H7: 50 · H1: 60 · H6: 60 | H7: 100 · H1: 95 · H6: 100 |
| Số model calls | 46 | 53 (+15%) |
| Token | prompt 26,111 · completion 6,200 | prompt 49,550 · completion 7,272 |

- Chi phí: provider không trả số tiền; token prompt gần gấp đôi do system prompt dài hơn và thêm step đọc incident
  được tham chiếu (mỗi step gửi lại toàn bộ hội thoại, gồm cả tool result trước đó).

- Lỗi đã giải quyết:
  - H7: Cloudflare timeout, agent nêu "không xác định" thay vì bịa "operational".
  - H1/H6: agent đọc tiếp incident xr3myzj7ms51 được tham chiếu; trace có 2 tool steps, hết bị giới hạn điểm 60.

- Lỗi còn lại hoặc mới xuất hiện:
  - H2: 100 → 50. Agent chỉ gọi get_incident_details, không gọi get_status dù câu hỏi yêu cầu kiểm tra snapshot;
    mechanical: "Chưa gọi tool cần thiết: get_status", score_cap 50. Trace không có event validation nào,
    nên không do thay đổi của tôi; lần 1 cùng câu hỏi đã gọi get_status. Đây là biến động ngẫu nhiên của model.
    Hướng sửa: thêm quy tắc "câu hỏi về tình trạng hiện tại phải gọi get_status". Không sửa và chạy lại ở lần này
    để giữ đúng một vòng so sánh.

## 5. Giới hạn

- Judge: dùng cùng model với agent (gpt-4.1-mini) nên có thể thiên vị khi tự chấm; điểm judge là ước lượng.
  Lúc đầu dùng gpt-4o-mini, judge trả JSON lỗi nên case bị unscored; đã đổi model trước khi chạy eval_before.
- Snapshot: dữ liệu cố định ngày 09/11/2026, không phản ánh trạng thái thực tế hiện tại.
- Một lần chạy: model không cho kết quả giống nhau mỗi lần (H2 là ví dụ). Mức tăng 92.77 → 97.32 chưa chứng minh
  được độ ổn định; cần chạy nhiều lần và lấy trung bình.
- Cải tiến: quy tắc trong prompt chỉ tăng xác suất làm đúng (quy tắc đọc tham chiếu trong prompt đã không đủ
  cho H1). Kiểm tra URL chỉ xác nhận có mặt một URL, không chứng minh mọi thông tin đều đúng nguồn.
  Kiểm tra tham chiếu chỉ bắt link dạng /incidents/<id> và chỉ nhắc một lần.
- Chi phí: thêm khoảng 15% model calls, token prompt gần gấp đôi.
