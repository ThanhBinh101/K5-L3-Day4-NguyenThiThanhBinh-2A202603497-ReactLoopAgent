# Evidence và hướng dẫn checkpoints

- [CP1.md](CP1.md): setup, cấu hình API, khám phá _load/execute và giới hạn dữ liệu.
- [CP2](../CP2.md): agent loop từng bước; [log thật](cp2_run.json) do người dùng cung cấp.
- [CP3.md](CP3.md): xử lý lỗi, step budget và trace.
- [CP4.md](CP4.md): hướng dẫn từng bước chạy eval, đọc điểm/trace, xử lý 429 và chuẩn bị CP5.
- [cp1_tools.json](cp1_tools.json): output tools mới chạy offline trên snapshot.
- [cp3_error_examples.json](cp3_error_examples.json): 8 tình huống mô phỏng offline.
- [cp3_tests.txt](cp3_tests.txt): kết quả 37 tests offline pass.

Evidence được ghi ngày 11/11/2026. Snapshot có captured_at riêng; không coi đó là
trạng thái live ngày ghi tài liệu. Không gọi model thật thêm để tạo các file CP1/CP3.
Chưa xác minh git push; xem checklist trong CP1. Không có API keys trong các file này.

- [CP4-quota.md](CP4-quota.md): eval lần 1 bị HTTP 429; 11/22 case được chấm, chưa có điểm tổng.
- [cp4_eval_before_quota.json](cp4_eval_before_quota.json): bản sao nguyên trạng của lần eval bị giới hạn API.
