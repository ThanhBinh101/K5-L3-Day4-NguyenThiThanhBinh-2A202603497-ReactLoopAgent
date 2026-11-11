# CP5 — Phân tích E2, cải tiến prompt và kiểm tra lại

## Mục tiêu và trạng thái

Chọn một lỗi có evidence, giải thích nguyên nhân có thể, sửa agent, chạy tests
và đối chiếu kết quả. CP5 đã có thay đổi cùng kiểm tra E2 sau sửa; CP4/CP6 full
suite chưa hoàn tất. Không gọi API thêm khi ghi tài liệu này.

## Bước 1 — Giữ bằng chứng baseline

- [cp4_eval_before_quota.json](cp4_eval_before_quota.json): report before, 11/22 case
  scored; phần còn lại bị 429, điểm tổng null.
- [cp5_E2_before.json](cp5_E2_before.json): E2 trích nguyên từ baseline, 89 điểm.
- [agent_before_cp5.py](agent_before_cp5.py): code trước thay đổi prompt.

Không sửa report, fixtures, eval cases, scenarios, grading hay rubric để tăng điểm.

## Bước 2 — Chọn lỗi E2 và đối chiếu trace

Câu hỏi về cache purge Cloudflare và website CLB non-FedRAMP.
Before tìm đúng incident `wp49bp8bqo6c`, nhưng suy đoán overall major có thể
liên quan incident Chicago đã resolved. Trace chỉ cho thấy hai dữ kiện cùng
xuất hiện, không xác nhận quan hệ nhân quả. Agent không đọc chi tiết Chicago.
Đây là lỗi được chọn; không coi toàn bộ answer sai hoặc chỉ sửa theo điểm judge.

## Bước 3 — Đặt giả thuyết

Model có thể biến các dữ kiện xuất hiện trong cùng output thành lời giải thích
nhân quả. Prompt cảnh báo về nguyên nhân lỗi người dùng nhưng chưa cụ thể về
quan hệ giữa overall status và một incident được liệt kê. Đây là giả thuyết hành
vi, không phải truy cập được suy nghĩ nội bộ model.

## Bước 4 — Sửa một quy tắc trong SYSTEM_PROMPT

Trong `../agent.py` thêm:

```text
Do not infer which incident causes the provider's overall status merely because
both appear in the same tool result. State such a relationship only when the
returned evidence explicitly supports it; otherwise say the cause of the overall
status is undetermined. A resolved incident alone does not explain an outage.
```

Không hard-code Chicago/incident ID/câu trả lời E2; không đổi loop hay dataset.
Kỳ vọng: ngừng suy đoán hoặc nói chưa xác định nguyên nhân nếu thiếu evidence.

## Bước 5 — Chạy tests offline

```bash
python3 -m unittest discover -s tests -v
```

37 tests pass sau sửa, output [cp5_tests.txt](cp5_tests.txt). Tests xác nhận hành
vi code được kiểm tra, không đo chất lượng prompt/model thật.

## Bước 6 — Chạy lại E2 qua API

```bash
python3 evals/run_evals.py --case E2 --output evidence/cp5_E2_debug.json
```

Lần kiểm tra mới nhất đã thành công: E2 scored 95/100, agent completed.
File [cp5_E2_debug.json](cp5_E2_debug.json) hiện chứa kết quả thành công này;
không còn là output 429 trước đó vì đã ghi đè cùng đường dẫn. Không tạo lại hay
khẳng định đã lưu một bản riêng lỗi retry cũ nếu không có file đó. Evidence 429
của full baseline vẫn giữ trong report CP4.

## Bước 7 — So sánh số liệu thật

| Chỉ số E2 | Before | After |
|---|---|---|
| Điểm judge | 89 | 95 (+6 điểm) |
| Model calls agent | 3 | 3 |
| Tool calls | 2 | 2 |
| Tokens agent | 4.425 | 8.214 |
| Tokens judge | 4.071 | 2.356 |
| Tổng tokens | 8.496 | 10.570 |
| Chi phí provider báo | 0 | 0 |

Config lưu trong hai report giống nhau: provider/judge openrouter,
model/judge openrouter/free, max_steps 8, dataset core. Dữ liệu so sánh máy đọc
được: [cp5_E2_comparison.json](cp5_E2_comparison.json).

After không còn nhắc suy đoán Chicago. Có timestamp snapshot, nguồn incident,
phạm vi FedRAMP và lưu ý website có thể lỗi do nguyên nhân khác. Điểm tăng trong
lần này, nhưng tokens tăng khoảng 24,4%; không thể nói tối ưu chi phí/tokens.

## Bước 8 — Đọc giới hạn, không chỉ điểm

- Judge nhận xét answer đúng nhưng dài, có phần phụ không cần thiết.
- Answer có captured_at nhưng vẫn dùng “hiện tại/đang diễn ra”: chưa nhất quán
  với bằng chứng snapshot cũ. Có từ lẫn ngôn ngữ cần biên tập.
- Không bỏ mọi giới hạn dữ liệu chỉ để theo gợi ý rút gọn của judge; giữ giới hạn
  quan trọng để người đọc không hiểu sai status live hay phạm vi khách hàng.
- Một lần chạy/case không chứng minh prompt mới luôn tốt hơn. Free router có
  thể chọn model nền khác nhau, kể cả khi report ghi cùng model ID.
- 95 là điểm E2, không phải điểm toàn bộ CP6. CP4 before vẫn thiếu 11 case.

## Bước 9 — Cập nhật NOTES và chuẩn bị CP6

[NOTES.md](../NOTES.md) đã ghi phân tích, thay đổi, số liệu thật và giới hạn.
Khi có quota, hoàn tất baseline đủ 22 case với code before và chạy full after
cùng cấu hình. Lưu các phiên bản code/report riêng, không sửa hay ghép điểm thủ công.

## Cách giải thích khi demo

“Em chọn E2 vì agent suy đoán quan hệ giữa overall outage và Chicago mà trace
không chứng minh. Em thêm quy tắc yêu cầu bằng chứng trực tiếp. Tests offline
37/37 pass; E2 sau sửa không còn suy đoán này và tăng từ 89 lên 95 điểm. Tuy nhiên
tokens tăng, vẫn còn diễn đạt chưa nhất quán về snapshot, và một lần chạy với
free router chưa chứng minh ổn định. Em chưa báo điểm toàn bộ CP6.”
