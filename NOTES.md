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

- Sửa gì, ở đâu:
- Vì sao chọn thay đổi này:

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
