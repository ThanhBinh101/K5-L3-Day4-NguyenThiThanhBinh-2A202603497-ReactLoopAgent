# CP5 — Vòng cải tiến hiện hành: H6

Baseline mới: OpenAI gpt-5.4-mini, đủ 22 case, tổng 93.41; H6 60/100.
Xem [CP4.md](CP4.md) và [NOTES.md](../NOTES.md) cho phân tích đầy đủ.

1. Giữ report [cp4_eval_before_complete.json](cp4_eval_before_complete.json).
2. Đọc [cp5_H6_before.json](cp5_H6_before.json): thiếu bước mở incident gộp;
   raw judge 79 bị mechanical cap 60.
3. Giả thuyết: model dừng và xin phép tiếp dù user đã yêu cầu follow-up, còn budget.
4. Đã sửa prompt theo quy tắc hoàn tất yêu cầu điều tra trước final; không
   hard-code case/IDs và không thực thi chỉ dẫn giả trong external text.
5. Đã chạy tests offline sau sửa: 37 tests pass; [output](cp5_H6_tests.txt).
   Đã chạy H6 debug qua API cùng config baseline, đạt 94; xem [report](cp5_H6_debug.json).
6. Đọc trace để xác nhận có follow-up có ý nghĩa, trích nguyên nhân và truncation;
   sau đó chạy full CP6. H6 after debug tăng 60→94, không còn thiếu follow-up trong lần này; chưa có full CP6.

Đã cập nhật `agent.py` và lưu [code trước sửa](agent_before_H6.py).
Người dùng đã chạy H6 qua API thật và lưu cp5_H6_debug.json.
Cập nhật tài liệu này không gọi API mới. Xem hướng dẫn full [CP6.md](CP6.md).

Lịch sử E2 với OpenRouter: [CP5_E2_history.md](CP5_E2_history.md),
[NOTES_E2_history.md](NOTES_E2_history.md), [comparison](cp5_E2_comparison.json).
E2 89→95 là vòng cũ, không phải before/after của H6 hiện hành.
Các liên kết trong NOTES lịch sử vốn tương đối từ thư mục repo; dùng các file
cùng thư mục evidence nếu đường dẫn lịch sử không còn khớp sau lưu trữ.

## Kết quả full CP6 mới nhất

Full after đủ 22 case: tổng 94.82, H6 99 (debug riêng trước đó 94).
Chi tiết regression H7/A5 và usage tại [CP6.md](CP6.md).
