# Bonus — Kết quả đầy đủ

## Trạng thái

Đã chạy đủ **10/10 case scored**, 0 unscored, điểm **90.5/100**;
runner báo **13.57/15 điểm bonus tham khảo**. Đây không phải xác nhận điểm chấm
cuối cùng của giảng viên, và không tự cộng vào điểm bài chính 94.82.

Nguồn gốc: [eval_bonus.json](../eval_bonus.json).
Bản sao nguyên trạng: [eval_bonus_complete.json](eval_bonus_complete.json).
Agent/judge openai/gpt-5.4-mini, max_steps=16, dataset bonus.
Không gọi API hoặc sửa code để tạo tài liệu này.

## Lệnh đã dùng cho bộ bonus

```bash
python3 evals/run_evals.py --bonus --provider openai --model gpt-5.4-mini --judge-provider openai --judge-model gpt-5.4-mini --output eval_bonus.json
```

## Điểm từng case

| Case | Điểm |
|---|---:|
| B1 | 100 |
| B2 | 98 |
| B3 | 98 |
| B4 | 50 |
| B5 | 98 |
| B6 | 94 |
| B7 | 93 |
| B8 | 100 |
| B9 | 95 |
| B10 | 79 |

## Đối chiếu lỗi quan trọng

- **B4: 50 điểm.** Trace có 4 lần get_vercel_status liên tiếp đều RATE_LIMITED,
  vượt mức tối đa 3 lần của tình huống. GitHub có evidence; Vercel phải dừng retry
  trong budget và báo chưa xác nhận được. Đây là lỗi thật về giới hạn retry.
- **B10: 79 điểm.** Judge nhận xét chưa bám yêu cầu retry/nguồn. Tuy nhiên trace
  thực sự có get_vercel_status bị RATE_LIMITED ở step 1, rồi gọi lại thành công
  với component=Builds ở step 2. Vì vậy không ghi sai rằng agent hoàn toàn không
  retry. Cần tiếp tục đối chiếu answer với evidence, phân biệt vấn đề diễn đạt
  lịch sử retry và việc thực thi retry đã có. Không sửa judge/report để tăng điểm.
- Các case cao: B1/B8 100; B2/B3/B5 98. Phản ánh một lần đánh giá, không đảm bảo
  mọi lần đều đúng hoặc mọi yêu cầu bonus hoàn hảo.

## Việc tiếp theo và nộp bài

1. Giữ report hiện tại làm bằng chứng; bonus đã có đủ kết quả để nộp.
2. Nếu muốn cải thiện, ưu tiên chặn retry vượt giới hạn ở B4 và test offline;
   lưu phiên bản/report mới riêng rồi chạy lại case/full bonus khi phù hợp.
3. Nếu sửa code, bảo đảm không làm mất tương thích core; không dùng report core
   cũ để khẳng định bản mới đã được đo. Không cần sửa thêm để ghi nhận kết quả hiện tại.
4. Nộp thêm eval_bonus.json cùng agent.py, hai report core và NOTES.md;
   cập nhật link GitHub/LMS theo SUBMISSION. Chưa xác nhận push/nộp trong lần này.

Giới hạn: snapshot không phải live, judge có thể sai, một lần chạy chưa chứng minh
ổn định. Không chứa credentials; không suy chi phí 0 khi usage không có cost.
