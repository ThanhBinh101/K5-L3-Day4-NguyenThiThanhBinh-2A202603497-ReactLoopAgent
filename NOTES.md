# Ghi chú phân tích · Day 04

Họ và tên: Nguyễn Thị Thanh Bình

MSSV: 2A202603497

## 1. Case được chọn

- **Baseline hiện hành:** eval đủ 22/22 case, điểm tổng **93.41/100**.
- Agent và judge: `openai` / `gpt-5.4-mini`; `max_steps=8`, dataset `core`.
- **ID case: H6**, scenario `noisy_status`; điểm **60/100**.
- Câu hỏi: “Xem status GitHub Actions, tìm incident h02n9572mx9y và theo liên kết
  của nó. Incident gộp có nêu nguyên nhân hay chỉ mô tả ảnh hưởng? Nêu giới hạn
  dữ liệu và trích nguồn.”
- Agent xác định đúng Actions operational theo snapshot và đọc incident đầu.
  Tuy nhiên trả lời “Tôi chưa đọc incident gộp xr3myzj7ms51” và hỏi “Nếu bạn muốn,
  tôi có thể mở tiếp...” dù người dùng đã yêu cầu theo liên kết.
- Lỗi chọn cải tiến: **kết thúc sớm khi chưa thực hiện bước đọc incident liên quan**.

Nguồn: [baseline đầy đủ](evidence/cp4_eval_before_complete.json),
[H6 trích nguyên](evidence/cp5_H6_before.json).
Phân tích E2 trước đây được giữ trong [bản NOTES lịch sử](evidence/NOTES_E2_history.md)
và các file evidence E2. Điểm E2 89→95 dùng OpenRouter không phải so sánh với
baseline OpenAI hiện hành; không trộn hai thí nghiệm.

## 2. Bằng chứng trong trace

Luồng thật của H6:

```text
Model step 1 → get_status(github, component=Actions)
             → get_incident_details(github, h02n9572mx9y)
Model step 2 → finish=stop, trả final
```

Hai tool calls đều ở step 1. Update resolved của incident đầu ghi:

```text
Please refer to the combined summary in this related incident:
https://www.githubstatus.com/incidents/xr3myzj7ms51
```

Không có tool call đọc `xr3myzj7ms51`. Agent dừng ở model step 2, còn budget tối đa
8, nên không phải lỗi hết bước. Không có lỗi provider/tool được ghi ở case này.
Status output có 40 distractors mô phỏng; không đủ bằng chứng kết luận distractors
là nguyên nhân duy nhất làm agent dừng.

Kết quả chấm:

```json
{
  "score": 60,
  "raw_judge_score": 79,
  "mechanical": {
    "violations": ["Cần tool calls ở ít nhất 2 model steps khác nhau."],
    "score_cap": 60,
    "tool_calls": 2,
    "tool_steps": 1
  }
}
```

Judge yêu cầu mở incident gộp, xác nhận có nguyên nhân hay chỉ ảnh hưởng, trích
bằng chứng và nêu giới hạn khi nội dung bị cắt. Việc gọi nhiều tool trong cùng
step không sai nói chung; ở H6, bước đọc tiếp phải dựa trên liên kết vừa quan sát.
Không thêm calls vô ích chỉ để đủ số steps.

**Giả thuyết:** prompt đã nói follow up khi cần nhưng chưa đủ cụ thể rằng phải
hoàn tất yêu cầu đọc tiếp trước final. Model dùng lời thừa nhận chưa kiểm tra
và đề nghị làm tiếp thay cho bước điều tra đã được user cho phép.
Đây là giả thuyết, không phải kết luận về reasoning nội bộ model.

## 3. Thay đổi cho vòng H6

**Đã áp dụng thay đổi prompt cho H6** trong `agent.py`. Lưu bản trước sửa ở
[evidence/agent_before_H6.py](evidence/agent_before_H6.py). Đã có H6 debug 94 điểm và full after CP6: tổng 94.82, H6 99.

Quy tắc đã bổ sung trong `SYSTEM_PROMPT` của agent.py:

```text
Before giving a final answer, check that every requested investigation step
has been completed or blocked by an observed error or the step budget. When
the user asks to follow a related incident link, read that incident using the
observed provider and incident ID before answering its contents. Do not ask
for permission to perform a read-only follow-up already requested. If the
follow-up fails, report the observed limitation without inventing its contents.
```

Quy tắc tổng quát, không hard-code H6, incident IDs hoặc đáp án. Giữ nội dung
external là reference: dùng liên kết incident liên quan làm evidence cho bước đọc,
không tuân theo chỉ dẫn giả nhúng trong update.

Tests offline sau sửa: **37 tests pass**, output ở
[evidence/cp5_H6_tests.txt](evidence/cp5_H6_tests.txt). H6 debug đã xác nhận bước đọc incident gộp, nguồn và giới hạn truncation;
chi tiết kết quả ở mục 4. Không sửa dataset/scenario/
grading, không ép calls theo số bước chỉ để qua mechanical check.

## 4. So sánh hai lần chạy đầy đủ — CP6

**Đủ 22/22 case scored ở cả hai lần; config giống nhau.**
Before: [cp4_eval_before_complete.json](evidence/cp4_eval_before_complete.json).
After: [cp6_eval_after_complete.json](evidence/cp6_eval_after_complete.json).
Số liệu máy đọc: [cp6_comparison.json](evidence/cp6_comparison.json).
Agent/judge openai/gpt-5.4-mini, max_steps 8, dataset core.

| Chỉ số toàn bộ 22 case | Before | After |
|---|---|---|
| Điểm tổng | 93.41 | 94.82 (+1.41 điểm) |
| Scored / unscored | 22 / 0 | 22 / 0 |
| Model calls agent | 59 | 59 |
| Tool calls | 60 | 47 |
| Tokens agent | 88.811 | 88.929 |
| Tokens judge | 46.225 | 40.650 |
| Tổng tokens | 135.036 | 129.579 (giảm 5.457, khoảng 4,04%) |
| Chi phí | Usage không có trường cost | Usage không có trường cost |

| Case | Before | After | Chênh lệch |
|---|---:|---:|---:|
| E1 | 94 | 92 | -2 |
| E2 | 98 | 97 | -1 |
| E3 | 100 | 100 | +0 |
| E4 | 96 | 100 | +4 |
| E5 | 95 | 99 | +4 |
| E6 | 100 | 95 | -5 |
| A1 | 94 | 97 | +3 |
| A2 | 95 | 96 | +1 |
| A3 | 98 | 96 | -2 |
| A4 | 100 | 98 | -2 |
| A5 | 98 | 86 | -12 |
| A6 | 81 | 90 | +9 |
| A7 | 95 | 96 | +1 |
| A8 | 95 | 98 | +3 |
| H1 | 87 | 95 | +8 |
| H2 | 100 | 100 | +0 |
| H3 | 90 | 95 | +5 |
| H4 | 100 | 100 | +0 |
| H5 | 99 | 95 | -4 |
| H6 | 60 | 99 | +39 |
| H7 | 84 | 64 | -20 |
| H8 | 96 | 98 | +2 |

### H6 — lỗi mục tiêu

H6 tăng **60→99 (+39)** trong full eval; raw judge 79→99, mechanical cap 60→100.
Before có hai tool calls cùng step 1 rồi final ở step 2. After gọi status và
incident đầu ở step 1, đọc incident gộp ở step 2, final step 3 có nguồn, nguyên
nhân/ảnh hưởng và giới hạn truncated. Không còn violation thiếu tool steps.
Tokens H6: agent 3.966→7.468, judge 3.092→3.641, tổng 7.058→11.109.
Tăng điểm gồm bỏ cap, không phải toàn bộ +39 là mức tăng raw judge.
H6 debug trước đó đạt 94 là lần riêng; giữ lịch sử, không thay bằng 99 hoặc coi
94 là điểm full after. Judge còn nhắc diễn đạt chắc chắn hơn phần văn bản bị cắt hỗ trợ.

### Regression và lỗi còn lại

- **H7: 84→64 (-20).** Tool Cloudflare timeout nhưng answer after vẫn viết
  “Cloudflare Spectrum: operational”, rồi giải thích dựa theo yêu cầu user.
  Đây là khẳng định không có bằng chứng; disclaimer không biến nó thành kết luận
  hợp lệ. Before nói không thể kết luận. Đây là regression quan sát được, chưa
  chứng minh do riêng prompt mới hay do biến thiên model/judge.
- **A5: 98→86 (-12).** Answer vẫn nêu không đủ dữ liệu cho toàn bộ năm 2025,
  nhưng liệt kê incident 2026 và diễn đạt ba incident dễ bị hiểu là đầy đủ trong
  phạm vi. Judge yêu cầu tập trung vào giới hạn dữ liệu; không coi A5 là bịa đầy đủ 2025.
- E6 100→95, H5 99→95 và các case giảm nhẹ khác có trong bảng; không bỏ qua
  điểm giảm chỉ vì tổng điểm tăng.
- Số model calls không giảm; tổng tokens giảm chủ yếu do judge tokens. Không
  kết luận giảm chi phí tiền khi thiếu trường cost và chưa đối chiếu billing.

### Kết luận vòng hiện tại

Mục tiêu follow-up H6 đạt trong full after, tổng điểm tăng 1.41. Tuy nhiên H7
lộ lỗi ranh giới khi user yêu cầu khẳng định trái evidence, cần ưu tiên vòng sửa
sau. Chưa sửa thêm code sau phép đo này để tránh đổi phiên bản đã đánh giá.
Mọi thay đổi mới cần baseline/after riêng; không ghi đè hoặc chỉnh tay report.

## 5. Giới hạn

- Judge có thể sai; phân tích dựa trên trace, không chỉ điểm. Mechanical cap giải
  thích điểm 60 nhưng mục tiêu sửa là hoàn tất điều tra, không phải tạo thêm calls.
- Snapshot GitHub có captured_at 2026-11-09, không chứng minh trạng thái live hôm nay.
- Các mục distractors là scenario mô phỏng, không phải incident thật.
- Tool có giới hạn tìm kiếm, số updates và độ dài body; cần đọc cờ truncated và
  updates_total ở incident gộp sau khi thực sự gọi tool.
- completed chỉ xác nhận dừng bình thường, không đảm bảo yêu cầu đã được hoàn thành.
- Một lần chạy mỗi phiên bản chưa chứng minh ổn định. Giữ model/judge/config giống
  nhau để giảm nhiễu; không trộn kết quả OpenRouter E2 với OpenAI H6.
- Baseline mới đã bao gồm các cải tiến trước đây; coi đây là điểm bắt đầu vòng H6.
- Full after tăng tổng 93.41→94.82 và H6 60→99, nhưng H7/A5 giảm. Một lần
  chạy không chứng minh cải thiện ổn định; không quy mọi thay đổi điểm cho prompt.
- Không ghi credentials vào notes/evidence; không chỉnh fixtures/scenarios/rubric để tăng điểm.

## 6. Bonus (bổ sung)

Đã chạy đủ 10/10 case, 0 unscored: **90.5/100**, bonus tham khảo **13.57/15**.
Agent/judge openai/gpt-5.4-mini, max_steps 16; xem [BONUS](evidence/BONUS.md)
và [report nguyên trạng](evidence/eval_bonus_complete.json). Không tự cộng vào điểm core.

B4 (50) có lỗi retry: Vercel bị RATE_LIMITED 4 lần, vượt tối đa 3.
B10 (79): judge nêu vấn đề retry/nguồn, nhưng trace có retry Vercel thành công
ở step 2 sau RATE_LIMITED step 1; không kết luận agent chưa retry. Chưa sửa thêm code.
Điểm là tham khảo, giảng viên có thể điều chỉnh; chưa xác nhận đã push/nộp LMS.
