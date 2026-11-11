# Ghi chú phân tích · Day 04

Họ và tên: Nguyễn Thị Thanh Bình

MSSV: 2A202603497

## 1. Case được chọn

- ID case: **E2**, scenario `normal`.
- Câu hỏi: “Cloudflare cache purge có incident nào liên quan không? Một website CLB không thuộc FedRAMP có chắc bị ảnh hưởng không?”
- Điểm ở lần chạy 1: **89/100**, `grade.status = scored`.
- Agent đã tìm đúng incident `wp49bp8bqo6c`, đọc chi tiết, nêu trạng thái `identified`,
  dẫn đúng URL và nhận ra cập nhật loại trừ khách hàng non-FedRAMP.
- Đoạn answer được chọn phân tích: “Tổng thể trạng thái Cloudflare hiện tại là
  major với Partial System Outage, nhưng phần lớn có thể liên quan đến sự cố
  mạng ở Chicago đã được resolve trước đó.”
- Lỗi cụ thể: suy đoán quan hệ giữa overall status và incident Chicago khi tool
  result không xác nhận quan hệ đó. Không quy kết toàn bộ answer là sai.

Bằng chứng gốc: [eval trước khi sửa](evidence/cp4_eval_before_quota.json).
Case trích nguyên dữ liệu: [cp5_E2_before.json](evidence/cp5_E2_before.json).

## 2. Bằng chứng trong trace

Các trường sau được trích từ trace thật E2; chỉ lược những trường không liên quan:

```json
{
  "name": "get_status",
  "arguments": { "provider": "cloudflare" },
  "result": {
    "captured_at": "2026-11-09T03:50:47.638054+00:00",
    "overall": { "indicator": "major", "description": "Partial System Outage" },
    "incidents": [
      {
        "id": "x9orajjmvvwa",
        "name": "Network Performance Issues in Chicago",
        "status": "resolved",
        "updated_at": "2026-11-07T05:35:16.013Z"
      }
    ]
  }
}
```

Trace chỉ có hai tool calls: `get_status(cloudflare)` và
`get_incident_details(cloudflare, wp49bp8bqo6c)`. Không có lời gọi đọc chi tiết
incident Chicago, cũng không có trường nào xác nhận Chicago gây ra overall `major`.
Ngay cả nếu đọc thêm chi tiết, vẫn cần bằng chứng trực tiếp cho quan hệ đó.

Chi tiết purge ghi `status = identified`, component `CDN Cache Purge`, và update
ban đầu: “Non-fedramp customers are not impacted.” Đây là bằng chứng loại trừ
non-FedRAMP **khỏi incident cụ thể này**, không đảm bảo website không thể lỗi vì nguyên nhân khác.

**Giả thuyết nguyên nhân:** model liên kết các dữ kiện xuất hiện trong cùng kết
quả thành một lời giải thích nhân quả. Prompt đã cảnh báo không quy incident
thành nguyên nhân lỗi người dùng, nhưng chưa nêu rõ không suy ra incident nào
gây overall status chỉ từ việc chúng xuất hiện cùng nhau. Đây là giả thuyết về
hành vi quan sát được, không phải kết luận về suy nghĩ nội bộ của model.

**Đối chiếu judge:** judge nhận xét cách nói `identified` và khuyên tránh suy đoán.
Answer đã nêu identified và không nói resolved cho incident purge; do đó không
coi riêng từ “xác nhận” là lỗi chính. Ưu tiên lỗi Chicago có evidence rõ hơn.

## 3. Thay đổi đã thực hiện

- File: [agent.py](agent.py), phần `SYSTEM_PROMPT`, mục Evidence and workflow.
- Thêm một quy tắc tập trung vào cùng một giả thuyết:

```text
Do not infer which incident causes the provider's overall status merely because
both appear in the same tool result. State such a relationship only when the
returned evidence explicitly supports it; otherwise say the cause of the overall
status is undetermined. A resolved incident alone does not explain an outage.
```

- Mục đích: khi có overall major và một incident resolved nhưng không có quan hệ
  được xác nhận, agent nói chưa xác định được nguyên nhân thay vì suy đoán.
- Quy tắc mang tính tổng quát, không hard-code Chicago, incident ID hay đáp án E2.
- Giữ nguyên logic loop, kiểm tra nguồn và các comments hướng dẫn ban đầu.
- Không sửa fixtures, dataset, scenarios, grading, rubric hoặc điểm eval.
- Lưu bản code trước thay đổi tại [agent_before_cp5.py](evidence/agent_before_cp5.py).
- Tests offline sau thay đổi: **37 tests pass**; output tại
  [cp5_tests.txt](evidence/cp5_tests.txt). Tests này không đo hiệu quả prompt với model thật.

## 4. So sánh hai lần chạy

**Đã chạy lại E2 sau cải tiến: 95/100, tăng 6 điểm so với 89/100.**
Nguồn after: [cp5_E2_debug.json](evidence/cp5_E2_debug.json), `scored`, agent `completed`.
Đây là kiểm tra một case, không phải `eval_after.json` đủ 22 case cho CP6.
Baseline CP4 vẫn chỉ chấm 11/22 case vì 429, điểm tổng `null`.

| Chỉ số | Before E2 | After E2 |
|---|---|---|
| Điểm E2 | 89/100 | 95/100 (+6 điểm) |
| Model calls của agent | 3 | 3 |
| Tool calls | 2 | 2 |
| Tokens agent | 4.425 | 8.214 |
| Tokens judge | 4.071 | 2.356 |
| Tổng tokens agent + judge | 8.496 | 10.570 (+2.074, khoảng 24,4%) |
| Chi phí provider báo | Agent 0; judge 0 | Agent 0; judge 0 |
| Điểm tổng toàn bộ 22 case | null, mới chấm 11/22 | Chưa có full after |

Bảng máy đọc được: [cp5_E2_comparison.json](evidence/cp5_E2_comparison.json).
Config được lưu trong hai report giống nhau: agent/judge `openrouter/free`,
provider `openrouter`, `max_steps=8`, dataset `core`. Không đảm bảo cùng model nền
vì dùng free router.

- **Hành vi đã quan sát tốt hơn:** answer after không còn suy đoán incident
  Chicago gây overall outage. Có nêu `captured_at`, dẫn đúng URL incident, tách
  phạm vi non-FedRAMP và không loại trừ các nguyên nhân khác gây lỗi website.
- **Nhận xét judge:** reason xác nhận tìm đúng incident, dẫn update phù hợp và
  không khẳng định nguyên nhân lỗi website. Judge đánh giá có thông tin phụ không
  cần thiết và đề nghị tóm tắt ngắn hơn.
- **Lỗi còn lại:** dù có captured_at, answer vẫn mở đầu “hiện tại” và nhắc
  “đang diễn ra” ở các đoạn sau. Cần diễn đạt nhất quán theo snapshot, không ngầm
  khẳng định trạng thái live. Có từ lẫn ngôn ngữ như “incidente”, “buiten range”.
- **Hiệu quả:** số calls không giảm; tổng tokens tăng. Không kết luận cải tiến
  tiết kiệm tokens hay chi phí dựa trên giá provider báo bằng 0.
- **Kết luận có giới hạn:** một lần E2 after không có lỗi Chicago và được judge
  chấm cao hơn. Chưa chứng minh prompt mới là nguyên nhân duy nhất hay ổn định
  trên mọi lần chạy/case. Không coi lỗi đã bị loại bỏ vĩnh viễn.

Lệnh đã dùng để kiểm tra E2:

```bash
python3 evals/run_evals.py --case E2 --output evidence/cp5_E2_debug.json
```

Bước tiếp theo cho CP4/CP6: hoàn tất baseline đủ bộ với code before đã lưu rồi
chạy full after cùng cấu hình để so sánh. Lưu bản after hiện tại trước nếu phục
hồi code before; không ghi đè mất thay đổi. Không sửa tay hoặc ghép điểm để tạo
report đầy đủ giả. Khi có `eval_after.json`, bổ sung số liệu toàn bộ, không dùng
95 điểm E2 làm điểm tổng 22 case.

## 5. Giới hạn

- LLM judge có thể bỏ sót hoặc đánh giá sai. Điểm 89 là đánh giá tham khảo; phân
  tích lỗi dựa trên đối chiếu answer với trace và tool evidence.
- Dữ liệu snapshot có captured_at riêng, không chứng minh trạng thái live hôm nay.
- Tool tìm theo chuỗi trong tập incident đã tải, trả tối đa 5 mục khớp; một số
  update có thể bị lược hoặc cắt. Không tìm thấy không chứng minh không tồn tại.
- Một lần chạy mỗi phiên bản không chứng minh độ ổn định hay quan hệ nhân quả
  giữa prompt mới và thay đổi điểm.
- `openrouter/free` có thể route tới model nền khác nhau; giữ cùng router ID chưa
  đảm bảo cùng model nền. Muốn so sánh chặt chẽ hơn cần model cố định khả dụng và
  baseline tương ứng; thay provider/model phải ghi rõ.
- Lần before chỉ chấm 11/22 case vì 429, chưa có điểm tổng. After mới có E2
  đạt 95 điểm; chưa có full-suite after. Chưa thể kết luận chất lượng tổng thể
  tăng hoặc chi phí giảm; tổng tokens E2 thực tế tăng.
- Không ghi API key hoặc credentials vào trace, notes và evidence.
