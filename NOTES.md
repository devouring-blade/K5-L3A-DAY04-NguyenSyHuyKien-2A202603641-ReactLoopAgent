# Ghi chú phân tích · Day 04

Họ và tên:
MSSV:

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

| Chỉ số | `eval_before.json` | `eval_after.json` |
|---|---|---|
| Điểm tổng |  |  |
| Điểm case đã chọn |  |  |
| Số model calls |  |  |
| Token, chi phí nếu có |  |  |

- Lỗi đã giải quyết:
- Lỗi còn lại hoặc mới xuất hiện:

## 5. Giới hạn

Nêu giới hạn của điểm judge, của snapshot và của việc chỉ chạy một lần mỗi phiên bản.
