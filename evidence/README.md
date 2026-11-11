# Evidence và hướng dẫn checkpoints

## Tài liệu hiện hành

- [CP1.md](CP1.md): setup, API, _load/execute và giới hạn dữ liệu.
- [CP2.md](CP2.md): giải thích agent loop; [log thật](cp2_run.json).
- [CP3.md](CP3.md): xử lý lỗi, budget và trace.
- [CP4.md](CP4.md): **đã chấm đủ 22/22, 93.41/100**, chọn H6 phân tích tiếp.
- [CP5.md](CP5.md): đã sửa prompt H6, 37 tests pass; H6 debug tăng 60→94.
- [NOTES.md](../NOTES.md): case hiện hành **H6**, score 60, raw judge 79/cap 60.
- [Baseline đầy đủ](cp4_eval_before_complete.json): report OpenAI gpt-5.4-mini nguyên trạng.
- [H6 before](cp5_H6_before.json): case trích nguyên từ report mới.

## Bằng chứng bổ trợ và lịch sử

- [cp1_tools.json](cp1_tools.json): output tools offline.
- [cp3_error_examples.json](cp3_error_examples.json): lỗi mô phỏng offline.
- [cp3_tests.txt](cp3_tests.txt): kết quả tests offline.
- [CP4-quota.md](CP4-quota.md), [report quota](cp4_eval_before_quota.json),
  [hướng dẫn CP4 cũ](CP4_partial_history.md): lần OpenRouter 11/22 case, hiện là lịch sử.
- [CP5 E2 lịch sử](CP5_E2_history.md), [NOTES E2 lịch sử](NOTES_E2_history.md): vòng cũ.
- [E2 comparison](cp5_E2_comparison.json), [E2 after](cp5_E2_debug.json): 89→95
  trên OpenRouter, không dùng làm so sánh cho baseline OpenAI mới.

Không trộn các cấu hình hay tự điền kết quả after chưa chạy. Snapshot có timestamp
riêng, không phải trạng thái live ngày ghi tài liệu. Không chứa API keys.

- [CP6.md](CP6.md): full after đủ 22/22, 94.82; so sánh H6 và regression H7/A5.
- [cp5_H6_debug.json](cp5_H6_debug.json): H6 sau sửa, 1/1 scored, 94 điểm.

- [cp6_eval_after_complete.json](cp6_eval_after_complete.json): full after nguyên trạng.
- [cp6_comparison.json](cp6_comparison.json): điểm từng case và usage before/after.

- [CP7.md](CP7.md): demo H6/H7, kiểm tra tên repo, commit/push và tự nộp LMS; chưa xác nhận hoàn tất nộp bài.
- [cp7_preflight.json](cp7_preflight.json), [cp7_tests.txt](cp7_tests.txt): kiểm tra local mới, 37 tests pass.
