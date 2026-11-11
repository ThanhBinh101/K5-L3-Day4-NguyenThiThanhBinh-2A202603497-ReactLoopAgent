# CP1 — Setup và khám phá tools

## Mục tiêu và trạng thái

CP1 yêu cầu repo riêng đúng tên, cấu hình provider/model/key, chạy thử hai tools,
và hiểu giới hạn dữ liệu. Chưa cần tự viết agent loop hay chạy bộ eval.

| Yêu cầu | Bằng chứng | Trạng thái |
|---|---|---|
| Repo riêng đúng tên | Remote origin trỏ tới repo ThanhBinh101/K5-L3-Day4-NguyenThiThanhBinh-2A202603497-ReactLoopAgent; upstream trỏ repo lớp | Đã kiểm tra remote local; không xác nhận quyền sở hữu chỉ từ remote |
| API chạy được | [cp2_run.json](cp2_run.json): log thật người dùng cung cấp, status completed | Đã có bằng chứng thành công trước đây; không phải kiểm tra key mới |
| Hai tools hoạt động | [cp1_tools.json](cp1_tools.json): output mới chạy offline | Đã kiểm tra |
| Push lên repo | Chưa thực hiện hay xác minh push trong lần ghi evidence này | Cần tự xác nhận |

## Bước 1 — Chuẩn bị repo

Trong thư mục repo, kiểm tra:

```bash
git remote -v
```

`origin` là nơi bạn push bài; `upstream` là repo đề. URL remote đúng không tự
chứng minh có quyền push. Sau khi tự kiểm tra nội dung cần nộp, commit và push
lên branch phù hợp để xác nhận. Không add `.env` hoặc API key.

## Bước 2 — Nạp cấu hình API

`.env` chứa cấu hình riêng tư. Ví dụ tên biến, không phải credentials thật:

```bash
PROVIDER=openrouter
MODEL=openrouter/free
OPENROUTER_API_KEY=your_actual_key
```

Code không tự đọc `.env`. Mỗi terminal mới cần:

```bash
set -a
source .env
set +a
```

`set -a` export các biến để Python con đọc được bằng `os.getenv`.
Nếu gặp lỗi certificate, dùng CA bundle đã cài, không tắt SSL verification:

```bash
export SSL_CERT_FILE="$(python3 -m certifi)"
```

Nếu chưa có certifi: `python3 -m pip install certifi`.
Lỗi 401 là vấn đề xác thực, không phải bằng chứng service GitHub gặp sự cố.

## Bước 3 — Hiểu `_load()`

`StatusTools(mode='snapshot')` đọc JSON từ `fixtures/`; không gọi model/API status.
`_load('github')` trả captured_at, source và dữ liệu summary/incidents; kết quả
được cache trong instance. Đây là method nội bộ, agent gọi qua tool registry.

```bash
python3 - <<'PY'
from status_tools import StatusTools
tools = StatusTools('snapshot')
data = tools._load('github')
print(data['captured_at'])
print(len(data['data']['incidents']['incidents']))
PY
```

Xem `loaded_incident_count` trong cp1_tools.json để biết số mục thực sự đọc.
Không suy ra trạng thái live hôm nay từ timestamp snapshot cũ.

## Bước 4 — Hiểu `get_status` và `execute`

```bash
python3 - <<'PY'
import json
from status_tools import StatusTools
tools = StatusTools('snapshot')
result = tools.execute('get_status', {'provider': 'github', 'component': 'Actions'})
print(json.dumps(result, ensure_ascii=False, indent=2))
PY
```

Luồng: execute → kiểm tra tên tool → get_status → _load → lọc → trả dict.

Đọc `overall`, `components`, `incidents`, `captured_at`, `source` và `note`.
`matching_incidents_in_last_50` là số mục khớp trong tập đã load;
`incidents` chỉ trả tối đa 5 mục, không phải toàn bộ mục khớp.

## Bước 5 — Đọc chi tiết incident

```bash
python3 - <<'PY'
import json
from status_tools import StatusTools
tools = StatusTools('snapshot')
status = tools.execute('get_status', {'provider':'github', 'component':'Actions'})
if status.get('incidents'):
    incident_id = status['incidents'][0]['id']
    details = tools.execute('get_incident_details', {'provider':'github', 'incident_id':incident_id})
    print(json.dumps(details, ensure_ascii=False, indent=2))
PY
```

Tool trả status, components, timestamps, updates, updates_total và URL.
Có thể chỉ giữ hai update mới nhất và update đầu tiên; mỗi body tối đa 900 ký tự.
`truncated` báo body bị cắt; updates_total lớn hơn số updates trả về báo đã lược update.

## Bước 6 — Vì sao không tìm thấy chưa đủ kết luận không có sự cố?

Trong `../status_tools.py`, `relevant()` dùng `query in text.casefold()`:
đây là substring matching, không phải hiểu ngữ nghĩa. Cụm “xóa bộ nhớ đệm” có thể
không khớp “cache purge”. Dữ liệu cũng chỉ là tập incident đã tải, không phải toàn
bộ lịch sử. Tool note và GUIDE mô tả phạm vi 50 incident gần nhất; get_status
không tự cắt `[:50]`, mà lọc trên danh sách đã load rồi cắt kết quả `matches[:5]`.

Do đó: “không có mục khớp trong dữ liệu” khác với “dịch vụ chắc chắn không gặp sự cố”.

## Tự giải thích khi demo

“Em đã cấu hình provider và chạy được API. Em thử hai tool bằng snapshot không
cần model: get_status xem tổng quan, get_incident_details đọc timeline và scope.
execute là registry thực thi, _load đọc/cache dữ liệu. Em luôn kiểm tra captured_at
và giới hạn tìm kiếm trước khi kết luận.”

CP1 đủ khi đã có các bước trên, hiểu được ý nghĩa và tự xác nhận push thành công.
