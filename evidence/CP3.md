# CP3 — Xử lý lỗi và ghi trace

## Mục tiêu và bằng chứng

Agent phải kết thúc có kiểm soát hoặc trả lỗi tool cho model để xử lý tiếp.
Không coi lỗi lấy dữ liệu là bằng chứng provider đang outage.

- [cp3_tests.txt](cp3_tests.txt): output chạy mới toàn bộ tests offline, 37 tests pass.
- [cp3_error_examples.json](cp3_error_examples.json): 8 tình huống lỗi chạy mới
  với FakeProvider và snapshot tools. Đây là mô phỏng OFFLINE, không phải lỗi live mới.
- Code liên quan: `../agent.py` và `../status_tools.py`.

## Bước 1 — Giới hạn số model calls

```python
for step in range(1, max_steps + 1):
    response = provider.complete(messages, schemas)
```

Mỗi lần gọi model tốn một step; nhiều tool calls trong một response vẫn là một step.
Hết budget chưa có final hợp lệ thì trả `status='max_steps'`, không completed.
Ví dụ `cases.max_steps`: budget 1, model gọi tool và nhận kết quả nhưng chưa được
hỏi tiếp nên dừng. Đây không phải xác nhận câu trả lời cuối.

## Bước 2 — JSON arguments bị lỗi

```python
arguments = json.loads(call['arguments'])
```

Nếu chuỗi không parse được hoặc JSON không phải object (ví dụ `[]`), trả observation:

```json
{"ok": false, "error": "INVALID_ARGUMENTS"}
```

Không gọi tools.execute với arguments lỗi. Vẫn gửi tool result có đúng call ID,
để model có thể sửa ở step tiếp theo. Xem cases.invalid_json và cases.non_object_json.
Trong các ví dụ offline, FakeProvider trả final ở bước kế nên status vẫn completed:
điều này kiểm tra loop, không chứng minh model thật biết tự sửa.

## Bước 3 — Tool lỗi là observation

Tool không tồn tại, provider không hợp lệ hoặc timeout được đóng gói thành dict lỗi.
`tools.execute` xử lý các lỗi dự kiến; agent còn bắt exception bất ngờ của tool.

| Case trong JSON | Hành vi quan sát |
|---|---|
| unknown_tool | UNKNOWN_TOOL, không crash loop |
| invalid_provider | ValueError từ validation của tool |
| timeout | TimeoutError do StatusTools(fail=True) giả lập |

Kết quả được ghi vào trace và gửi cho model. Model có thể sửa tham số hoặc nói
chưa đủ dữ liệu. Không suy diễn “không lấy được GitHub status” thành “GitHub sập”.
Không yêu cầu đổi tên provider LLM khi lỗi nói về provider argument của status tool:
trong get_status, provider là github/cloudflare; ở run.py, provider là OpenRouter/OpenAI…

## Bước 4 — Model bị cắt output

Nếu `finish='length'`, model chưa hoàn tất. Code trả `status='error'` ngay,
không thực thi các calls trong response bị cắt, không gắn nhãn completed cho nội dung dở dang.
Xem cases.truncated_output.

## Bước 5 — Provider LLM lỗi

Không có phản hồi model thì kết thúc run với `status='error'`. Hàm
`_safe_provider_error` phân loại HTTP/SSL/connection/timeout từ exception chain,
không đưa key, HTTP response body hoặc exception text tùy ý vào trace.

Ví dụ cases.provider_http_401 tạo lỗi 401 giả lập, kết quả có thông báo xác thực.
Các mã như 402/429 chỉ là gợi ý kiểm tra; cần đối chiếu thông tin provider để biết
nguyên nhân cụ thể. API key nằm trong môi trường, không lưu vào evidence.

## Bước 6 — Đọc trace và usage

```json
{"event":"tool", "step":1, "name":"get_status",
 "arguments":{"provider":"github"}, "result":{}}
```

Đoạn trên là hình dạng minh họa, không phải output thật. Output đầy đủ nằm trong
cp3_error_examples.json.

- event=model: model step và finish reason.
- event=tool: tên, arguments, kết quả hoặc lỗi của một lần thực thi tool.
- event=error: lỗi làm run dừng.
- usage: một mục cho mỗi response model đã nhận; FakeProvider thường trả `{}`.
- validation_missing_source: kiểm tra nguồn bổ sung của Bài 4, ngoài phần lõi CP3.

## Bước 7 — Chạy lại tests

Từ thư mục repo:

```bash
PYTHONDONTWRITEBYTECODE=1 python3 -m unittest discover -s tests -v
```

Một số test quan trọng trong tests/test_lab.py:

| Test | Kiểm tra |
|---|---|
| test_invalid_json_sent_back_to_model | INVALID_ARGUMENTS được model nhận lại |
| test_max_steps | Budget kết thúc bằng max_steps |
| test_model_truncation_is_not_completed | length không phải completed |
| test_multiple_tool_calls | Mọi tool result có đúng ID |
| test_errors_are_observations | Tool lỗi trả dict thay vì crash |

37 tests pass bao gồm cả các phần khác và kiểm tra nguồn Bài 4, không phải 37 tests
chỉ dành cho CP3. Tests offline kiểm tra code, không chứng minh chất lượng model live.

## Cách giải thích khi demo

“Em tách lỗi tool và lỗi provider. Lỗi tool được trả về model như observation;
lỗi provider kết thúc run vì không thể lấy phản hồi. Em giới hạn số model calls,
không coi output bị cắt là hoàn thành, và ghi trace để biết step nào gọi tool gì,
arguments ra sao, kết quả hoặc lỗi thế nào.”
