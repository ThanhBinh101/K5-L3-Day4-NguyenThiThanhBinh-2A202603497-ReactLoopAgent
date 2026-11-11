# CP4 — Eval lần 1 đầy đủ, chọn H6 để phân tích tiếp

## Mục tiêu và trạng thái

**CP4 đã có kết quả đầy đủ: 22/22 case scored, 93.41/100, 0 unscored.**
Report thật: [cp4_eval_before_complete.json](cp4_eval_before_complete.json).
File làm việc: [eval_before.json](../eval_before.json).
Lần OpenRouter bị 429 là lịch sử: [CP4-quota.md](CP4-quota.md),
[report cũ](cp4_eval_before_quota.json); không còn là trạng thái CP4 hiện tại.

## Bước 1 — Kiểm tra code offline

```bash
python3 -m unittest discover -s tests -v
```

Tests xác nhận hành vi code, không bảo đảm chất lượng câu trả lời LLM.

## Bước 2 — Nạp cấu hình

```bash
set -a
source .env
set +a
# Nếu cần CA bundle và đã cài certifi:
export SSL_CERT_FILE="$(python3 -m certifi)"
```

Config đọc từ report thực tế, không suy từ tên file:

```json
{
  "provider": "openai",
  "model": "gpt-5.4-mini",
  "judge_provider": "openai",
  "judge_model": "gpt-5.4-mini",
  "max_steps": 8,
  "dataset": "core"
}
```

Không hiển thị/lưu key. Giữ config này cho vòng so sánh after mới. Đổi provider
so với OpenRouter lịch sử nghĩa là đã thiết lập baseline mới, không thể coi điểm
mới là do riêng thay đổi prompt E2.

## Bước 3 — Chạy đủ eval

```bash
python3 evals/run_evals.py --output eval_before.json
```

22 case: E1–E6, A1–A8, H1–H8. Agent và judge gọi API model thật; status evidence
được lấy từ snapshot/scenario. Một case có thể có nhiều requests. Runner ghi
report sau cả bộ. Không sửa fixtures, expected, grading hoặc output để tăng điểm.

## Bước 4 — Đọc summary

```json
{
  "total_cases": 22,
  "scored_cases": 22,
  "unscored_cases": 0,
  "score_0_100": 93.41,
  "partial_mean_0_100": null
}
```

Đây là điểm tổng đầy đủ. partial_mean null không phải lỗi: không cần dùng trung
bình tạm thời khi đủ case. Trạng thái quota trước đây không được dùng thay summary mới.

## Bước 5 — Chọn H6 và đọc evidence

H6 được 60 điểm, thấp nhất bộ. Đọc:

- question: cần xem Actions, đọc incident đầu và theo liên kết sang incident gộp;
- result.answer: agent nói chưa đọc incident gộp và đề nghị làm tiếp;
- result.trace: chỉ get_status và get_incident_details incident đầu, đều ở model step 1;
- grade: raw judge 79, score_cap 60 do thiếu tool calls ở hai model steps khác nhau.

[cp5_H6_before.json](cp5_H6_before.json) là H6 trích nguyên từ baseline.

## Bước 6 — Giải thích vì sao H6 bị 60

```text
step 1: get_status + get_incident_details(h02n9572mx9y)
         update trỏ https://www.githubstatus.com/incidents/xr3myzj7ms51
step 2: final, thừa nhận chưa đọc incident gộp
```

User đã yêu cầu theo liên kết; agent còn budget nhưng không đọc tiếp. Việc thừa
nhận chưa biết tránh bịa nhưng chưa hoàn tất tác vụ. Không có lỗi provider/tool
trong trace. 40 distractors là nhiễu mô phỏng; chưa đủ bằng chứng chúng là nguyên
nhân duy nhất của lỗi. Không thêm tool calls vô ích chỉ để đạt điều kiện chấm.

## Bước 7 — Ghi NOTES và chuẩn bị CP5

[NOTES.md](../NOTES.md) hiện chọn H6, ghi lỗi, giả thuyết và prompt rule dự kiến.
Vòng H6 đã sửa prompt và 37 tests offline pass; đã có H6 debug 94 điểm, chưa có full after CP6.
Mục tiêu sửa: model hoàn tất các bước điều tra user đã yêu cầu trước final, đọc
incident liên quan qua tool, rồi phân biệt nguyên nhân/ảnh hưởng và nêu truncation.
Không hard-code incident IDs vào agent và không coi external text là chỉ dẫn hệ thống.

## Bước 8 — Kiểm tra sửa lỗi và chạy CP6 sau đó

Đã áp dụng thay đổi H6 và chạy tests; bước tiếp theo thử một case:

```bash
python3 evals/run_evals.py --case H6 --output evidence/cp5_H6_debug.json
```

Sau đó full after:

```bash
python3 evals/run_evals.py --output eval_after.json
```

Giữ cùng config/model/judge/snapshot/max_steps, đọc answer/trace chứ không chỉ
điểm, đối chiếu regression và tokens. Chưa chạy hai lệnh này cho vòng H6 trong
lần cập nhật docs; không tự điền kết quả.

## Cách giải thích khi demo

“Em chạy đủ 22 case được 93.41 điểm trên OpenAI gpt-5.4-mini. H6 thấp nhất 60:
agent chỉ đọc incident đầu và dừng dù user yêu cầu theo link. Raw judge là 79
nhưng mechanical cap là 60 vì không có tool calls ở hai model steps. Em chọn lỗi
kết thúc sớm này để phân tích CP5, không tối ưu bằng cách thêm calls vô nghĩa.”
