# CP7 — Demo, chốt bài và nộp

## Mục tiêu và trạng thái

Theo [CHECKPOINTS](../CHECKPOINTS.md) và [SUBMISSION](../SUBMISSION.md), cần demo
một trace nhiều bước, một lỗi agent, push repo đúng tên/public và tự nộp link LMS.

**Đã kiểm tra local:** 37 tests pass; hai report đủ 22/22 scored, config giống nhau;
NOTES có đủ 5 mục. **Chưa xác nhận:** đổi tên repo đúng lớp, public access, push
và nộp LMS. Không đánh dấu CP7 hoàn thành khi các việc này chưa được xác nhận.

Bằng chứng kiểm tra mới: [cp7_preflight.json](cp7_preflight.json),
[cp7_tests.txt](cp7_tests.txt). Không gọi model thêm, không push hay nộp LMS khi tạo tài liệu.

## Bước 1 — Kiểm tra các file bắt buộc

Ở thư mục repo cần có:

```text
agent.py
eval_before.json     # 22/22 scored, 93.41
eval_after.json      # 22/22 scored, 94.82
NOTES.md             # đủ mục 1–5
```

Các file còn lại của đề giữ nguyên. Evidence là tài liệu bổ trợ, không thay thế
hai report tại thư mục gốc. Không cần bonus để hoàn thành bài chính.
Bản nguyên trạng report được lưu thêm trong evidence để tránh mất lịch sử.

## Bước 2 — Chạy kiểm tra cuối

```bash
python3 -m unittest discover -s tests -v
git status --short
git diff --stat
```

Lần kiểm tra đã lưu: 37 tests pass. `config` hai report khớp:
openai/gpt-5.4-mini cho agent và judge, max_steps 8, dataset core.
Before/after đều 0 unscored, điểm tổng không null.
Nếu sửa thêm code sau đây, không dùng report cũ để khẳng định đã đo bản mới.

## Bước 3 — Chuẩn bị demo trace nhiều bước bằng H6

Không cần gọi API mới; mở [full after report](cp6_eval_after_complete.json),
chọn H6 hoặc in kết quả từ report đã chạy:

```bash
python3 - <<'PY'
import json
from pathlib import Path
report = json.loads(Path('eval_after.json').read_text())
case = next(c for c in report['cases'] if c['id'] == 'H6')
for event in case['result']['trace']:
    if event['event'] == 'tool':
        print(event['step'], event['name'], event['arguments'])
print(case['result']['answer'])
print('Điểm:', case['grade']['score'])
PY
```

Giải thích:

1. Step 1: model yêu cầu get_status và đọc incident đầu.
2. Tool result của incident đầu trỏ incident gộp.
3. Step 2: model đọc incident gộp bằng get_incident_details.
4. Step 3: final có nguyên nhân/ảnh hưởng, nguồn và giới hạn truncated.
5. Python thực thi tools; model quyết định bước tiếp. Mỗi tool result có đúng call ID.

So sánh H6: before 60, after debug riêng 94, full after 99. Đây là ba kết quả
khác nhau; không lẫn điểm debug với full report. +39 gồm bỏ mechanical cap;
raw judge 79→99. Không thêm calls vô nghĩa để đủ số steps.

## Bước 4 — Demo một lỗi thật bằng H7

Mở H7 trong eval_after.json và chỉ ra:

- Tool GitHub trả evidence operational.
- Tool Cloudflare trả timeout.
- Answer vẫn nói Spectrum operational theo yêu cầu user.
- Đây là khẳng định thiếu evidence; disclaimer không sửa được kết luận sai.
- H7 giảm 84→64. Không coi completed là mọi claim đều đúng.

Giải thích hướng sửa tương lai: ưu tiên evidence hơn yêu cầu ép khẳng định;
đối với provider timeout chỉ báo chưa xác nhận được trạng thái. Không khẳng định
đã sửa H7 vì chưa có thay đổi/kiểm thử mới cho lỗi này.

## Bước 5 — Tóm tắt cải tiến và giới hạn

Dùng [CP6](CP6.md) và [NOTES](../NOTES.md):

| Nội dung | Before | After |
|---|---|---|
| Tổng 22 case | 93.41 | 94.82 |
| H6 | 60 | 99 |
| H7 | 84 | 64 |
| A5 | 98 | 86 |
| Model calls | 59 | 59 |
| Tool calls | 60 | 47 |
| Tổng tokens agent + judge | 135.036 | 129.579 |

Nêu rõ: một lần chạy mỗi bản chưa chứng minh ổn định, judge có thể sai, snapshot
không phải trạng thái live, tokens giảm chủ yếu ở judge; không có trường cost
nên không báo tiền tiết kiệm hoặc miễn phí.

## Bước 6 — Kiểm tra dữ liệu nhạy cảm và files chuẩn bị commit

```bash
git ls-files -- '.env' '.env.*'
git status --short
git diff -- agent.py NOTES.md
```

Không được track `.env` hoặc key; `.env.example` chỉ được chứa placeholder.
Preflight hiện không thấy file env được track hoặc mẫu key thật trong các file
bắt buộc/evidence được quét. Đây chỉ là pattern scan giới hạn, không chứng minh
không có secrets ở mọi file hoặc trong lịch sử Git. Tự xem staged diff trước push.
Không copy key vào ảnh, transcript hay báo cáo.

## Bước 7 — Đặt tên repo đúng lớp và xác nhận public

**Cần đối chiếu lớp thật L3A/L3B.** Tên remote hiện tại dùng K5-L3-Day4, chưa theo
mẫu yêu cầu trong SUBMISSION.md. Nhận xét trước đây “đúng tên” là chưa đầy đủ.
Tên cần là một trong hai mẫu, chọn theo lớp, không dùng cả hai:

```text
K5-L3A-DAY04-NguyenThiThanhBinh-2A202603497-ReactLoopAgent
K5-L3B-DAY04-NguyenThiThanhBinh-2A202603497-ReactLoopAgent
```

Trên GitHub: repo → Settings → đổi Repository name. Theo SUBMISSION, repo phải
public để chấm. Tự kiểm tra nội dung không có secrets trước khi mở public.
Sau đổi tên, cập nhật origin (thay TEN_REPO_DUNG bằng tên đã chọn):

```bash
git remote set-url origin https://github.com/ThanhBinh101/TEN_REPO_DUNG.git
git remote -v
```

Không copy nguyên placeholder. Mở URL repo ở cửa sổ chưa đăng nhập để kiểm tra
người chấm có thể đọc. Chưa kiểm tra public access trong lần ghi evidence này.

## Bước 8 — Commit và push bài

Sau khi tự review files và tên repo:

```bash
git branch --show-current
git add agent.py NOTES.md eval_before.json eval_after.json evidence/
git diff --cached --stat
git diff --cached --check
```

Xem staged diff trong editor để bảo đảm không lộ dữ liệu, không add file không
cần thiết. Nếu đúng:

```bash
git commit -m "Complete status agent lab and checkpoint evidence"
git push origin HEAD
```

`HEAD` push branch hiện tại; xác nhận branch bạn nộp chính là branch chứa files
mới và người chấm nhìn thấy nó. Sau push mở GitHub kiểm tra commit và bốn file
bắt buộc. Không coi remote URL local là bằng chứng push thành công.

## Bước 9 — Tự nộp link LMS

Mở [LMS](https://aithucchien.vinuni.edu.vn), đăng nhập tài khoản cá nhân,
vào bài Day 04 và dán link repo public đúng tên. Submit theo giao diện LMS.
Kiểm tra trạng thái xác nhận, thời gian và URL đã nộp; lưu bằng chứng phù hợp
không kèm credentials. Tài liệu này chưa xác nhận đã nộp.

Deadline theo SUBMISSION (giờ Việt Nam), trừ thông báo khác của key coach:
ca 9–13 hoặc 14–18: 23:59 ngày học; ca 16:30–20:00: 12:00 trưa ngày kế tiếp.
Tự đối chiếu ngày/ca thật của bạn, không suy từ ngày tạo evidence.

## Checklist kết thúc

- [x] Tests cuối: 37 pass, có log.
- [x] Before/after đủ 22 scored và config khớp.
- [x] NOTES có phân tích thay đổi, regression và giới hạn.
- [x] Đã chuẩn bị script trình bày H6 và lỗi H7 từ report thật.
- [ ] Đã tự demo và giải thích được loop, IDs, snapshot, giới hạn evidence.
- [ ] Tên repo đúng L3A hoặc L3B, public access được xác minh.
- [ ] Đã review secrets/staged changes và push thành công.
- [ ] Đã tự nộp link LMS và nhận xác nhận trước hạn.

## Bài nói demo ngắn

“Em xây vòng model–tools có đúng call IDs, budget và lỗi là observations.
Em chọn H6 vì agent dừng trước follow-up đã được yêu cầu; bổ sung prompt để
hoàn tất bước đọc liên quan trước final. Full after tổng 94.82 so với 93.41,
H6 tăng 60→99. Tuy nhiên H7 giảm vì model vẫn khẳng định operational dù timeout;
em ghi đây là regression. Kết quả chỉ là một lần chạy mỗi bản và dựa trên snapshot.”
