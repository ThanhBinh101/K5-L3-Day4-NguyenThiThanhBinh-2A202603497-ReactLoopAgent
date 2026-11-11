# CP4 — Chạy eval lần 1 và đọc kết quả

## Mục tiêu và trạng thái hiện tại

CP4 đo chất lượng agent trên 22 câu hỏi cố định và tìm lỗi để chuẩn bị cải tiến
ở CP5. Khác CP3, eval gọi model thật, gồm cả agent và LLM judge.

**Hiện chưa hoàn thành CP4:** đã chấm 11/22 case; 11 case còn lại lỗi HTTP 429.
Không có điểm tổng. Chưa phân biệt được giới hạn ngắn hạn hay hết quota ngày.

Bằng chứng:

- [cp4_eval_before_quota.json](cp4_eval_before_quota.json): bản sao nguyên trạng lần chạy thật.
- [CP4-quota.md](CP4-quota.md): ghi nhận chi tiết giới hạn API.
- [eval_before.json](../eval_before.json): file gốc ở thư mục repo, có thể bị ghi đè khi chạy lại.

## Bước 1 — Hiểu luồng đánh giá

```text
Case: question + scenario
  → run_agent với model thật và status tools dùng snapshot/scenario
  → answer + trace + usage
  → kiểm tra bằng code + LLM judge
  → grade của từng case
  → tổng hợp summary và ghi file JSON
```

Snapshot/scenario là bằng chứng về service status; API LLM chỉ phục vụ agent và
judge. Đừng nhầm snapshot với status live hôm chạy eval.
Expected/rubric dành cho chấm điểm, không được đưa đáp án expected cho agent.
Không sửa fixtures, cases, scenarios, grading hoặc kết quả eval để tăng điểm.

## Bước 2 — Kiểm tra offline trước

Từ thư mục repo:

```bash
python3 -m unittest discover -s tests -v
```

Lần kiểm tra đã ghi ở [cp3_tests.txt](cp3_tests.txt) có 37 tests pass.
Tests pass nghĩa là các hành vi code được test hoạt động; không đảm bảo mọi câu
trả lời model đúng và không đảm bảo quota đủ chạy eval.

## Bước 3 — Nạp cấu hình provider và model

```bash
set -a
source .env
set +a
# Nếu cần CA bundle và đã cài certifi:
export SSL_CERT_FILE="$(python3 -m certifi)"
```

Không in API key ra terminal hoặc chép vào evidence. `.env` phải chứa key hợp lệ
và tên biến đúng với provider. Source lại .env sẽ thay các biến shell đã export
nếu file cũng khai báo cùng biến.

Cấu hình thực tế trong lần chạy đã lưu:

```json
{
  "provider": "openrouter",
  "model": "openrouter/free",
  "judge_provider": "openrouter",
  "judge_model": "openrouter/free",
  "max_steps": 8,
  "dataset": "core"
}
```

Eval core mặc định dùng budget 8 model calls/case, khác mặc định 6 của run_agent.
Free router có thể chọn model nền khác nhau giữa các requests; giữ cùng tên router
không đảm bảo cùng model nền. Ghi nhận giới hạn này khi so sánh before/after;
nếu chọn model ID cố định mới thì thiết lập lại baseline cùng cấu hình đó.

## Bước 4 — Chạy eval lần 1

```bash
python3 evals/run_evals.py --output eval_before.json
```

Chạy đủ 22 case E1–E6, A1–A8, H1–H8. Agent có thể gọi model nhiều lần/case,
judge cũng gọi model, nên số requests nhiều hơn 22. Theo GUIDE, runner ghi kết
quả khi cả bộ kết thúc; đừng dừng giữa chừng nếu muốn giữ kết quả qua runner này.

Muốn debug riêng A6 sau khi API khả dụng:

```bash
python3 evals/run_evals.py --case A6 --output eval_A6_debug.json
```

File một case chỉ để debug; không phải kết quả đủ 22 case của CP4. Không tự ghép
hoặc sửa tay điểm vào report toàn bộ để biến thành kết quả đầy đủ.

## Bước 5 — Đọc summary trước

```bash
python3 - <<'PY'
import json
with open('eval_before.json', encoding='utf-8') as f:
    data = json.load(f)
print(json.dumps(data['summary'], ensure_ascii=False, indent=2))
PY
```

Kết quả thực tế đã lưu:

```json
{
  "total_cases": 22,
  "scored_cases": 11,
  "unscored_cases": 11,
  "score_0_100": null,
  "partial_mean_0_100": 95.18
}
```

95.18 chỉ là trung bình các case đã chấm, không phải điểm CP4 toàn bộ.
`null` nghĩa là chưa đủ dữ liệu để báo điểm tổng, không phải điểm 0.

## Bước 6 — Đọc từng case

Mỗi phần tử trong `cases` có các trường cần hiểu:

| Trường | Cách đọc |
|---|---|
| id, question, scenario | Case nào, yêu cầu gì, có scenario mô phỏng nào |
| expected | Tiêu chí cần đối chiếu, không phải lời giải để agent đọc |
| result.status | Agent hoàn thành, hết bước hay lỗi |
| result.answer | Câu trả lời thực tế |
| result.trace | Model đã gọi tool gì, args gì, nhận bằng chứng/lỗi nào |
| result.usage | Tokens/chi phí provider báo cho agent |
| grade.status | scored hay unscored |
| grade.score | Điểm từng case, có thể null |
| grade.reason | Giải thích vì sao được điểm đó |
| grade.improvement | Gợi ý cần đối chiếu với trace trước khi sửa |
| grade.mechanical | Vi phạm bằng code, trần điểm, số tool calls/steps |
| grade.judge_usage | Usage của judge, riêng với usage của agent |

Ví dụ thật E1 được 94 điểm. Judge gợi ý nêu rõ snapshot, nguồn chi tiết và giới hạn
dữ liệu. Đây là điểm khởi đầu để đọc trace, không phải kết luận judge luôn đúng.
`completed` chỉ xác nhận run kết thúc bình thường, không đảm bảo đủ bằng chứng
hoặc mọi claim đều chính xác.

## Bước 7 — Xác định nguyên nhân unscored

```bash
python3 - <<'PY'
import json
with open('eval_before.json', encoding='utf-8') as f:
    data = json.load(f)
for case in data['cases']:
    if case['grade']['status'] == 'unscored':
        print(case['id'])
        print('Agent error:', case['result'].get('error'))
        print('Grade reason:', case['grade'].get('reason'))
PY
```

Trong lần chạy này, A6, A7, A8 và H1–H8 đều có:

```text
Provider HTTP 429. Rate limit or quota reached. Wait before retrying and check provider limits.
```

Cả 11 run lỗi ngay step đầu, chưa gọi tool. Judge ghi chưa phân biệt được lỗi
agent và hạ tầng; không thể chấm năng lực trả lời trên các case này. Không quy
lỗi 429 thành trạng thái outage GitHub/Cloudflare.

## Bước 8 — Xử lý quota và chạy lại

1. Kiểm tra quota/rate limits của tài khoản OpenRouter.
2. Nếu là giới hạn tạm thời, đợi; nếu hết quota ngày thì đợi reset hoặc chọn
   cấu hình có quota phù hợp. Evidence hiện tại không xác định loại giới hạn.
3. Runner mặc định `--workers 1` đã chạy tuần tự; không tăng concurrency khi bị 429.
4. Giữ bản evidence lỗi, sau đó chạy lại đủ bộ khi API khả dụng.
5. Nếu đổi model/provider, ghi rõ và dùng cấu hình đó cho cả baseline lẫn lần sau.
   Không xem hai cấu hình khác nhau là so sánh chỉ thay prompt/code.

Trong thời gian chờ có thể đọc 11 case đã chấm để học; không báo đã hoàn thành CP4.

## Bước 9 — Chuẩn bị đầu vào CP5

Chọn case đã chấm có lỗi chất lượng và ghi nhận:

- ID, câu hỏi và điểm.
- Claim đáng nghi trong answer.
- Tool result/trace chứng minh thiếu, sai hoặc vượt phạm vi.
- Giả thuyết nguyên nhân: prompt, loop, hay giới hạn dữ liệu.
- Thay đổi dự định và cách kiểm tra.

Ví dụ hướng phân tích: model nói “hiện tại” dù evidence là snapshot cũ, hoặc dùng
updated_at làm ngày xảy ra sự cố. Chỉ ghi lỗi của một case nếu trace thật xác nhận.
Không điền NOTES.md bằng kết quả/so sánh chưa chạy. Code trước CP4 này đã có cải
tiến từ trước, nên coi bản hiện tại là baseline của vòng đánh giá tiếp theo.

## Điều kiện hoàn thành và cách giải thích khi demo

CP4 đạt khi có report đủ 22 case, tất cả được chấm, score_0_100 khác null,
và bạn đã đọc reason/mechanical/answer/trace để nêu được lỗi có bằng chứng.

Cách giải thích trạng thái hiện tại:

“Em đã chạy eval lần 1 với agent và judge dùng OpenRouter free router. Có 11 case
được chấm, trung bình tạm thời 95.18; 11 case còn lại lỗi HTTP 429. Điểm tổng null
nên em chưa coi CP4 hoàn thành. Em lưu report nguyên trạng trong evidence, kiểm
tra quota rồi chạy lại đủ bộ, sau đó chọn lỗi có trace để cải tiến ở CP5.”
