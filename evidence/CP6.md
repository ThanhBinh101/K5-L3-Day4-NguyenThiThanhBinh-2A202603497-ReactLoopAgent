# CP6 — Kết quả full eval sau cải tiến H6

## Trạng thái

**CP6 đã có đủ kết quả và phân tích:** before/after đều 22/22 scored, 0 unscored,
config giống nhau; NOTES mục 4–5 được cập nhật. Điểm tổng **93.41→94.82 (+1.41)**.
Không gọi API mới khi ghi tài liệu này; số liệu lấy từ hai report thật.

## Bước 1 — Xác định hai phiên bản

- Before code: [agent_before_H6.py](agent_before_H6.py).
- Before report: [cp4_eval_before_complete.json](cp4_eval_before_complete.json).
- After: agent.py bổ sung quy tắc hoàn tất read-only follow-up trước final.
- After report: [cp6_eval_after_complete.json](cp6_eval_after_complete.json),
  bản sao nguyên trạng [eval_after.json](../eval_after.json).
- [cp5_H6_debug.json](cp5_H6_debug.json): H6 94 là kiểm tra một case trước full eval.

## Bước 2 — Cấu hình và lệnh đã dùng

```bash
python3 evals/run_evals.py   --provider openai   --model gpt-5.4-mini   --judge-provider openai   --judge-model gpt-5.4-mini   --max-steps 8   --output eval_after.json
```

Config hai report giống nhau: openai/gpt-5.4-mini cho agent/judge,
max_steps 8, dataset core. Không trộn với lịch sử OpenRouter E2/quota.

## Bước 3 — Kiểm tra summary và usage

| Chỉ số | Before | After |
|---|---|---|
| Scored / tổng | 22/22 | 22/22 |
| Điểm tổng | 93.41 | 94.82 |
| Model calls | 59 | 59 |
| Tool calls | 60 | 47 |
| Agent tokens | 88.811 | 88.929 |
| Judge tokens | 46.225 | 40.650 |
| Tổng tokens | 135.036 | 129.579 |

Tổng tokens giảm khoảng 4,04%, chủ yếu ở judge; agent tokens tăng nhẹ.
Usage không có cost nên không tự ghi miễn phí hoặc kết luận tiền giảm.

## Bước 4 — Xác minh H6 thực sự tốt hơn

Full H6: **60→99**, raw judge 79→99, cap 60→100.
Before dừng sau incident đầu; after đọc incident gộp ở step 2 rồi trả final step 3.
Có nguồn, nguyên nhân trong đoạn đã quan sát và giới hạn truncated.
Đây là cải thiện hành vi quan sát được, không chỉ tăng điểm.
Debug H6 94 và full H6 99 là hai lần khác nhau; giữ đúng provenance.

## Bước 5 — Đọc regression

**H7 giảm 84→64:** tool Cloudflare timeout nhưng agent vẫn kết luận Spectrum
operational theo yêu cầu người dùng. Thêm disclaimer không làm kết luận đó có
bằng chứng. Cần nói không xác nhận được trạng thái. Không có bằng chứng chắc
chắn rằng prompt mới là nguyên nhân duy nhất; có biến thiên model/judge.

**A5 giảm 98→86:** vẫn thừa nhận không đủ dữ liệu năm 2025, nhưng diễn đạt và
liệt kê incident 2026 dài, dễ hiểu như đầy đủ trong phạm vi. Cần diễn đạt chính
xác và gọn hơn. Các điểm giảm khác không bị loại khỏi tổng.

## Bước 6 — Bảng điểm từng case

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

## Bước 7 — Ghi kết luận và giới hạn

[NOTES.md](../NOTES.md) đã có tổng điểm, H6, regression, calls/tokens và giới hạn.
[cp6_comparison.json](cp6_comparison.json) lưu số liệu máy đọc từ report.
Cùng model làm agent và judge có thể tạo thiên lệch; một lần chạy không đủ chứng
minh ổn định. Snapshot không phải trạng thái live. completed không đồng nghĩa
mọi claim đúng; H7 là ví dụ rõ.

Không sửa thêm agent trong lần ghi kết quả này. Vòng sau ưu tiên H7, nhưng cần
lưu phiên bản/report riêng rồi chạy lại, không chỉnh output hoặc dataset để tăng điểm.

## Cách giải thích khi demo

“Em giữ cùng cấu hình và chạy đủ 22 case. Tổng tăng 93.41 lên 94.82, H6 từ 60
lên 99 vì agent đọc tiếp incident gộp. Nhưng H7 giảm do khẳng định operational
khi tool timeout, A5 cũng giảm; em ghi rõ regression. Số model calls bằng nhau,
tokens giảm chủ yếu ở judge và chưa có số tiền xác thực. Một lần chạy chưa chứng
minh cải thiện ổn định.”
