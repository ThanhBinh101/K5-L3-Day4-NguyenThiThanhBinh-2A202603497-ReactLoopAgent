# CP2 — Hoàn thiện agent loop

## Kết quả kiểm tra

- `agent.py` đã implement `run_agent(question, provider, tools, max_steps=6)`.
- Log thật do người dùng cung cấp được trích nguyên phần JSON vào `evidence/cp2_run.json`.
- Log này có `status=completed`, 3 model calls và 4 tool calls; không phải một lần chạy API mới.
- Kiểm tra offline: `PYTHONDONTWRITEBYTECODE=1 python3 -m unittest discover -s tests -v`: 37 tests pass.
- Code còn có xử lý lỗi (CP3) và kiểm tra nguồn (Bài 4); không cần bỏ các phần này để đạt CP2.

## Bước 1 — Chuẩn bị messages và khai báo tools

Trong `run_agent`:

```python
schemas = getattr(tools, 'schemas', SCHEMAS)
prompt = getattr(tools, 'system_prompt', SYSTEM_PROMPT)
messages = [{'role': 'system', 'content': prompt},
            {'role': 'user', 'content': question}]
```

`system` đặt vai trò và quy tắc; `user` chứa câu hỏi. `schemas` mô tả tên tool,
arguments và kiểu dữ liệu cho model. Schema không tự thực thi tool.
`getattr` dùng cấu hình của tools nếu có, ngược lại dùng mặc định của bài chính.

## Bước 2 — Gọi model

```python
response = provider.complete(messages, schemas)
```

Một lần gọi model là một step. Response có `message`, `calls`, `finish`, `usage`.
Model có thể yêu cầu nhiều tool trong cùng một response hoặc trả câu trả lời cuối.
`usage` lưu riêng theo từng lần gọi model để xem token và chi phí nếu provider trả về.

## Bước 3 — Giữ nguyên assistant message

```python
messages.append(message)
```

Message chứa các tool calls với ID tương ứng. Không thay message bằng riêng phần
text vì sẽ mất ngữ cảnh model đã yêu cầu tool nào và kết quả nào đang được chờ.

## Bước 4 — Parse arguments và thực thi mọi tool call

```python
for call in calls:
    arguments = json.loads(call['arguments'])
    observation = tools.execute(call['name'], arguments)
```

Ví dụ model gửi arguments là chuỗi `'{"provider":"github"}'`; `json.loads`
chuyển thành dict `{'provider': 'github'}` cho registry `execute`.
Code thật còn kiểm tra dict và xử lý lỗi; đoạn trên chỉ minh họa luồng thành công.
Model chọn tool; Python mới là phần thực thi tool. Phải xử lý hết calls trước
lần gọi model kế tiếp, không chỉ xử lý call đầu tiên.

## Bước 5 — Trả kết quả với đúng tool_call_id

```python
messages.append({
    'role': 'tool',
    'tool_call_id': call['id'],
    'content': json.dumps(observation, ensure_ascii=False),
})
```

Nếu model gọi hai tools có ID `a` và `b`, mỗi kết quả phải gắn đúng ID của nó.
ID nối kết yêu cầu và kết quả, không phải tên tool hay incident ID.
Sau khi mọi kết quả được append, vòng lặp gọi model lại với lịch sử đã bổ sung.

## Bước 6 — Nhận câu trả lời cuối

Khi `finish == 'stop'` và không có calls, agent lấy `message['content']`.
Nếu kiểm tra nguồn đạt, trả `completed`. Không coi output bị cắt (`length`) là hoàn thành.
Hàm luôn trả các trường `status`, `answer`, `trace`, `usage`; có thể thêm `error`.

## Bước 7 — Đọc bằng chứng lần chạy thật

Trong `evidence/cp2_run.json`:

| Step model | Hành động |
|---|---|
| 1 | Gọi `get_status(provider='github')`. |
| 2 | Gọi ba lần `get_incident_details` để đọc các incident. |
| 3 | Trả câu trả lời có nguồn và `finish='stop'`. |

Bốn lần thực thi tool không phải bốn model steps. `completed` chứng minh luồng
kết thúc bình thường, không chứng minh mọi nhận định trong answer đều chính xác.
Dữ liệu là snapshot ngày 2026-11-09, không phải kiểm tra trạng thái live hôm nay.
Còn điểm cần cải thiện: chưa lọc Actions từ đầu, có incident API Requests ngoài
phạm vi chính, và chưa nói rõ update bị cắt. Đây là dữ liệu phân tích cho CP5.

## Bước 8 — Tự chạy lại và giải thích khi demo

Từ thư mục repo, với `.env` đã có provider, model và key hợp lệ:

```bash
set -a
source .env
set +a
# Nếu cần xử lý lỗi CA và đã cài certifi:
export SSL_CERT_FILE="$(python3 -m certifi)"
python3 run.py 'GitHub Actions hiện có sự cố không? Hãy kiểm tra incident gần đây.'
```

Tự kiểm tra `status=completed`, có tool events, có answer và source URLs.
Không cần chạy eval đầy đủ để hoàn thành CP2.

Cách giải thích ngắn: “Agent gửi câu hỏi và schema cho model. Khi model yêu cầu
tool, Python parse arguments, thực thi mọi call, rồi trả JSON với đúng call ID.
Model đọc observations và tiếp tục cho tới câu trả lời cuối hoặc hết budget.”
