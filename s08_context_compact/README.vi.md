# s08: Thu Gọn Ngữ Cảnh: Dọn Chỗ Trống Trước Khi Đầy Bộ Nhớ

[English](README.md) · [中文](README.zh.md) · [日本語](README.ja.md) · [Tiếng Việt](README.vi.md)

s01 → s02 → s03 → s04 → s05 → s06 → s07 → `s08` → [s09](../s09_memory/) → s10 → ... → s16 → s17

> *"Context will fill up, so the Harness needs a way to make room."* (Ngữ cảnh rồi sẽ đầy, nên Harness cần có cơ chế dọn chỗ trống). Bốn bước được thực hiện tuần tự từ chi phí thấp đến chi phí cao.
>
> **Lớp Harness**: Thu gọn (Compaction) giữ cho cửa sổ ngữ cảnh có hạn luôn hữu ích trong suốt quá trình thực hiện một tác vụ dài.


Trong quá trình Agent làm việc, mọi nội dung đọc file, kết quả câu lệnh và phản hồi của mô hình đều được lưu trữ trong danh sách `messages`. Lịch sử này dần dần sẽ vượt quá kích thước cửa sổ ngữ cảnh của mô hình.

Bài học này xây dựng một đường ống thu gọn ngữ cảnh gồm bốn bước. Đường ống này ưu tiên cắt giảm các kết quả công cụ có thể khôi phục lại được, và chỉ thực hiện tóm tắt toàn bộ lịch sử khi những phương pháp cắt giảm đó vẫn chưa đủ.

![Context Compact overview](images/compact-overview.en.svg)


## Hiểu Đúng Về Ngữ Cảnh (Context)

Hãy hình dung cửa sổ ngữ cảnh giống như một cuốn sổ nháp tạm thời của mô hình. Các tin nhắn của người dùng, phản hồi của mô hình, khối `tool_use` và `tool_result` đều được ghi lần lượt vào đó theo thứ tự. Mô hình sẽ đọc lại toàn bộ nội dung đó mỗi khi tiếp tục giải quyết tác vụ.

Cuốn sổ nháp này có dung lượng giới hạn cố định. Khi một yêu cầu vượt quá dung lượng, API sẽ từ chối cuộc gọi với lỗi `prompt_too_long`. Trong các tác vụ lập trình, kết quả công cụ thường là thủ phạm ngốn nhiều dung lượng nhất:

- Đọc một file mã nguồn dài sẽ đưa toàn bộ nội dung file đó vào ngữ cảnh.
- Log chạy kiểm thử và build dự án có thể ngốn hàng chục kilobyte chỉ sau một lần chạy.
- Tìm kiếm trên nhiều file sẽ liên tục nối thêm vô số kết quả.

Khi tác vụ tiếp diễn, `messages` không ngừng phình to. Cơ chế thu gọn ngữ cảnh (compaction) giúp kiểm soát sự tăng trưởng này trong khi vẫn bảo toàn được mục tiêu hiện tại, các ràng buộc của người dùng và các công việc đang tiến hành.


## Vì Sao Kết Quả Công Cụ Được Ưu Tiên Xử Lý Trước?

Tóm tắt toàn bộ lịch sử có thể thu nhỏ kích thước rất nhanh, nhưng mỗi lần tóm tắt đều đánh mất một số chi tiết và tiêu tốn thêm một lượt gọi mô hình.

Kết quả công cụ là đối tượng lý tưởng hơn để xử lý trước tiên:

1. Kết quả đọc file lớn có thể được ghi ra đĩa và đọc lại khi cần.
2. Một câu lệnh cũ hoàn toàn có thể chạy lại để lấy kết quả mới.
3. Các kết quả gần đây nhất thường có giá trị liên quan cao hơn đối với bước thực hiện hiện tại.
4. Việc cắt tỉa văn bản và chỉnh sửa cấu trúc không tiêu tốn thêm lượt gọi mô hình nào.

Do đó, đường ống thu gọn tuân theo nguyên tắc tăng dần mức độ mất mát thông tin và chi phí: lưu trữ ra đĩa (persist), cắt gọt (trim), thay thế các kết quả cũ, và cuối cùng mới dùng đến tóm tắt (summarize).

![Four-step compaction pipeline](images/compaction-layers.en.svg)


## Bước 1: tool_result_budget

Một phản hồi của mô hình có thể yêu cầu gọi nhiều công cụ cùng lúc. Toàn bộ các khối `tool_result` sau khi hoàn thành sẽ được ghi chung vào một tin nhắn người dùng cuối cùng. Khi tổng dung lượng nội dung của chúng vượt quá `200_000` ký tự, `tool_result_budget` sẽ xử lý các kết quả lớn nhất trước.

Mỗi kết quả vượt quá ngưỡng `LARGE_RESULT_CHAR_LIMIT = 30000` sẽ được ghi nguyên văn ra file:

```text
.task_outputs/tool-results/<tool_use_id>.txt
```

Trong khi đó, ngữ cảnh chỉ giữ lại đường dẫn file kèm một đoạn trích xem trước 2.000 ký tự:

![Persisting large results](images/layer1-budget.en.svg)

Vòng lặp cốt lõi lưu trữ các kết quả theo thứ tự kích thước giảm dần:

```python
blocks = [block for block in content
          if isinstance(block, dict)
          and block.get("type") == "tool_result"]
total = sum(len(str(block.get("content", ""))) for block in blocks)

ranked = sorted(
    blocks,
    key=lambda block: len(str(block.get("content", ""))),
    reverse=True,
)
for block in ranked:
    if total <= max_chars:
        break
    content = str(block.get("content", ""))
    if len(content) <= self.LARGE_RESULT_CHAR_LIMIT:
        continue
    block["content"] = self.persist_large_output(
        block.get("tool_use_id", "unknown"), content)
    total = sum(len(str(item.get("content", ""))) for item in blocks)
```

Bước này chỉ kiểm tra nhóm kết quả công cụ mới nhất vừa sinh ra. Toàn bộ kết quả đầy đủ vẫn có thể truy cập được tại đường dẫn đã lưu, do đó lưu trữ ra đĩa là thao tác an toàn nhất để chạy đầu tiên.


## Bước 2: snip_compact

Khi số lượng tin nhắn trong lịch sử vượt quá 50, `snip_compact` sẽ ghi toàn bộ lịch sử ra `.transcripts/`, sau đó giữ lại 3 tin nhắn đầu tiên và 46 tin nhắn gần đây nhất. Một dấu mốc lưu trữ (archive marker) sẽ lấp vào vị trí còn lại, ghi nhận số lượng tin nhắn đã cắt bỏ và trỏ tới file bản ghi đầy đủ.

```python
head_end = 3
tail_start = len(messages) - (max_messages - head_end - 1)

if self.has_tool_use(messages[head_end - 1]):
    while (head_end < tail_start
           and self.is_tool_result(messages[head_end])):
        head_end += 1

if (tail_start > 0
        and self.is_tool_result(messages[tail_start])
        and self.has_tool_use(messages[tail_start - 1])):
    tail_start -= 1

transcript = self.write_transcript(messages)
marker = {"role": "user", "content":
          f"[{tail_start - head_end} messages archived at {transcript}]"}
messages = [*messages[:head_end], marker, *messages[tail_start:]]
```

Các điểm cắt được tính toán để bảo vệ các cặp `assistant(tool_use)` và `user(tool_result)`. Một kết quả công cụ mồ côi (không có yêu cầu gọi công cụ tương ứng) sẽ khiến yêu cầu API tiếp theo trở nên không hợp lệ.

Bước này kiểm soát số lượng tin nhắn. Tuy nhiên, các kết quả công cụ nằm trong những tin nhắn được giữ lại vẫn có thể rất dài.


## Bước 3: micro_compact

Sau hai bước đầu tiên, hàm `prepare` sẽ ước tính dung lượng ngữ cảnh còn lại và chỉ kích hoạt `micro_compact` khi dung lượng vượt quá `CONTEXT_CHAR_LIMIT`. Trong số các kết quả mà mô hình đã đọc qua, `micro_compact` giữ lại 3 kết quả mới nhất và rút gọn các kết quả cũ dài hơn 120 ký tự cho đến khi ngữ cảnh giảm về khoảng 80% hạn mức. Trước khi thay thế một kết quả cũ, nó luôn ghi nội dung đầy đủ ra đĩa để đảm bảo mọi điểm rút gọn đều có thể phục hồi:

![Replacing old results with recovery paths](images/micro-compact.en.svg)

```python
unseen = self.unseen_tool_result_positions(messages)
consumed = [entry for entry in results if entry[:2] not in unseen]

for _, _, block in consumed[:-self.KEEP_RECENT_RESULTS]:
    if self.estimate_chars(messages) <= target_chars:
        break
    content = str(block.get("content", ""))
    if len(content) <= 120:
        continue
    saved_path = self.persisted_output_path(content)
    if not saved_path:
        saved_path = self.save_output(block["tool_use_id"], content)
    block["content"] = f"[Earlier tool result saved at {saved_path}]"
```

Các kết quả mới thông thường vẫn được giữ nguyên vẹn cho đến khi mô hình đọc qua chúng. Nếu riêng một đợt kết quả chưa đọc đã quá lớn so với ngữ cảnh, hàm `fit_tool_results` sẽ lưu các kết quả lớn nhất ra đĩa và chỉ giữ lại đoạn xem trước 1.000 ký tự kèm đường dẫn tới nội dung đầy đủ. Điều này giúp tránh việc phải tóm tắt toàn bộ lịch sử trước khi mô hình kịp xem qua kết quả mới.

Hai bước đầu tiên chạy ở mọi vòng lặp. Bước 3 chỉ chạy khi ngữ cảnh vượt quá giới hạn. Cả ba bước đều là các thao tác xử lý cấu trúc và văn bản có tính tất định và có thể khôi phục hoàn toàn; chúng không làm phát sinh thêm lượt gọi API nào.


## Bước 4: compact_history

Sau khi chạy `micro_compact` và `fit_tool_results`, hệ thống tiếp tục ước lượng dung lượng ngữ cảnh bằng hàm `estimate_chars(messages)`:

```python
CONTEXT_CHAR_LIMIT = 50000

def estimate_chars(messages):
    return len(json.dumps(messages, default=str, ensure_ascii=False))
```

Khi số lượng ký tự vẫn vượt quá ngưỡng `CONTEXT_CHAR_LIMIT`, `compact_history` sẽ thực hiện bốn việc:

1. Ghi toàn bộ lịch sử tin nhắn ra thư mục `.transcripts/`.
2. Yêu cầu mô hình tạo một bản tóm tắt trạng thái thực tế.
3. Giữ yêu cầu ban đầu của người dùng tách biệt khỏi phần tóm tắt đó.
4. Thay thế toàn bộ lịch sử hiện tại bằng một tin nhắn `[Compacted]` duy nhất.

![History summary](images/auto-compact.en.svg)

```python
def compact_history(messages, active_request):
    transcript = self.write_transcript(messages)
    print(f"[transcript saved: {transcript}]")
    summary = self.summarize_history(messages)
    return [self.summary_message(
        "Compacted", active_request, summary, transcript)]
```

Lệnh gọi tóm tắt yêu cầu mô hình ghi lại mục tiêu, các file liên quan, các quyết định đã đưa ra, công việc còn lại và ràng buộc của người dùng mà không thực thi lại các chỉ dẫn trong lịch sử. Giao diện CLI truyền `active_request` vào Vòng lặp Agent vì kết quả công cụ cũng sử dụng `role=user`. Một tin nhắn sau thu gọn sẽ lưu yêu cầu này dưới mục `Current user request`, đặt bản tóm tắt dưới mục `Conversation summary`, và kèm theo đường dẫn bản ghi đầy đủ.

Bài học này sử dụng số lượng ký tự làm ngưỡng kích hoạt, và mọi ngưỡng giới hạn liên quan đều dùng chung đơn vị này.


## Vì Sao Thứ Tự Này Là Cố Định?

Đường ống tuân thủ nghiêm ngặt thứ tự này và chỉ bước vào giai đoạn tóm tắt gây mất mát thông tin khi thực sự bắt buộc:

```python
messages = self.tool_result_budget(messages)
messages = self.snip_compact(messages)
if self.estimate_chars(messages) > self.CONTEXT_CHAR_LIMIT:
    target = int(self.CONTEXT_CHAR_LIMIT * 0.8)
    messages = self.micro_compact(messages, target)
    if self.estimate_chars(messages) > self.CONTEXT_CHAR_LIMIT:
        messages = self.fit_tool_results(messages, target)
    if self.estimate_chars(messages) > self.CONTEXT_CHAR_LIMIT:
        messages = self.compact_history(messages, active_request)
```

Thứ tự này thỏa mãn hai ràng buộc quan trọng:

1. Bước 1 và 2 chạy ở mọi vòng. Bước 3 chỉ chạy khi vượt ngưỡng, và duy nhất Bước 4 phát sinh thêm yêu cầu gọi API.
2. Mọi kết quả công cụ bị rút ngắn đều lưu lại đường dẫn đáng tin cậy bên trong `.task_outputs/tool-results/`; chỉ khi tình trạng tràn bộ nhớ vẫn tiếp diễn thì mới chuyển sang tóm tắt lịch sử bằng mô hình.

Do đó, mỗi vòng lặp luôn khởi đầu bằng thao tác có chi phí thấp nhất và thông tin dễ khôi phục nhất.


## Khôi Phục Khi Bị API Từ Chối

Đếm số lượng ký tự chỉ là một cách ước tính gần đúng số token mô hình sử dụng. API vẫn có khả năng trả về lỗi `prompt_too_long`. Khi đó, `reactive_compact` sẽ lưu lại bản ghi, tóm tắt lịch sử cũ hơn và giữ lại 5 tin nhắn gần nhất:

```python
tail_start = max(0, len(messages) - self.KEEP_RECENT_MESSAGES)
if (tail_start > 0
        and self.is_tool_result(messages[tail_start])
        and self.has_tool_use(messages[tail_start - 1])):
    tail_start -= 1

old_history = messages[:tail_start] if tail_start else messages
summary = self.summarize_history(old_history)
message = self.summary_message(
    "Reactive compact", active_request, summary, transcript)
messages = [message, *messages[tail_start:]] if tail_start else [message]
```

Điểm cắt cũng ngăn chặn việc tách rời lệnh gọi công cụ khỏi kết quả tương ứng của nó, trong khi `active_request` bảo toàn nguyên vẹn yêu cầu hiện tại của người dùng. Giá trị `MAX_REACTIVE_RETRIES = 1` cho phép thử khôi phục một lần. Nếu vẫn xảy ra lỗi độ dài ngữ cảnh lần thứ hai, ngoại lệ sẽ được ném ra cho phía gọi xử lý.


## Tích Hợp Vào Vòng Lặp Agent

```python
def agent_loop(messages, active_request):
    while True:
        messages[:] = COMPACTOR.prepare(messages, active_request)

        try:
            response = client.messages.create(
                model=MODEL, system=SYSTEM, messages=messages,
                tools=TOOLS, max_tokens=8000)
            reactive_retries = 0
        except Exception as error:
            message = str(error).lower()
            too_long = ("prompt_too_long" in message
                        or "too many tokens" in message)
            if too_long and reactive_retries < MAX_REACTIVE_RETRIES:
                messages[:] = COMPACTOR.reactive_compact(
                    messages, active_request)
                reactive_retries += 1
                continue
            raise
```

Mọi lượt gọi mô hình đều đi qua cùng một đường ống chuẩn hóa. Sau khi nhận `query`, CLI gọi `agent_loop(history, query)`, nhờ đó việc thu gọn lặp đi lặp lại không làm mất yêu cầu hiện tại. Hệ thống chỉ yêu cầu tóm tắt khi `micro_compact` vẫn để ngữ cảnh vượt ngưỡng hoặc khi API từ chối tiếp nhận.


## Công Cụ compact Chủ Động

Ngưỡng tự động chỉ nhận biết được dung lượng của ngữ cảnh. Ngoài ra, mô hình cũng có thể chủ động gọi công cụ `compact` sau khi hoàn thành một giai đoạn công việc mà giai đoạn tiếp theo chỉ cần đến bản tóm tắt:

```python
{"name": "compact",
 "description": "Summarize earlier conversation to free context space."}
```

Một phản hồi có thể yêu cầu nhiều công cụ cùng lúc, chẳng hạn như ghi file rồi mới thu gọn ngữ cảnh. Harness sẽ thực thi đầy đủ toàn bộ loạt công cụ đó trước và nối thêm từng `tool_result` cho mỗi `tool_use`. Nó chỉ thực hiện tóm tắt sau khi lượt tương tác đó đã hoàn tất:

```python
tool_calls = [
    block for block in response.content if block.type == "tool_use"
]
results = []
compact_requested = False

for block in tool_calls:
    if block.name == "compact":
        output = "Compaction requested after this tool batch."
        compact_requested = True
    else:
        output = execute_tool(block)
    results.append({"type": "tool_result", "tool_use_id": block.id,
                    "content": output})

messages.append({"role": "user", "content": results})

if compact_requested:
    messages[:] = COMPACTOR.compact_history(messages, active_request)
```

Quy trình này đảm bảo không có kết quả công cụ nào bị mồ côi. Nó cũng lưu giữ lại dấu vết của các thao tác ghi file hoặc tác vụ phụ trước khi thu gọn, giúp mô hình không thực hiện lặp lại chúng.


## Những Gì Bài Học Này Bổ Sung

| Thành phần | Vòng lặp thực thi dùng chung | Bổ sung ở s08 |
| --- | --- | --- |
| Agent Loop | Gọi mô hình, chạy công cụ, nối kết quả | Chạy `COMPACTOR.prepare()` trước mỗi lần gọi mô hình |
| Hooks | Kiểm tra phân quyền, ghi log công cụ, xử lý kết quả | Giữ nguyên điểm kích hoạt thực thi công cụ |
| Ngữ cảnh (Context) | Nối thêm vào `messages` | Lưu kết quả lớn ra đĩa, lưu trữ lịch sử cũ, tóm tắt, và thử lại một lần khi gặp lỗi độ dài |
| Công cụ (Tools) | 5 công cụ cơ bản | Bổ sung thêm công cụ `compact`, tổng cộng 6 công cụ |

> **Ranh giới với bài s09:** s08 quản lý ngữ cảnh có hạn trong phiên làm việc hiện tại và có thể loại bỏ các chi tiết có thể khôi phục được. s09 sẽ lưu trữ những thông tin bắt buộc phải tồn tại qua các lần thu gọn và xuyên suốt các phiên làm việc tương lai.


## Thử Nghiệm

```bash
cd learn-claude-code
python s08_context_compact/code.py
```

### Thử nghiệm 1: Thay thế các kết quả cũ

```text
Read the README.md files from s01_agent_loop through s05_todo_write.
Compare their top-level headings and summarize the naming pattern.
```

Tác vụ này đọc ít nhất 5 file. Các kết quả mới thường vẫn giữ nguyên vẹn cho đến khi mô hình đọc qua một lần; một kết quả chưa đọc quá khổ sẽ chỉ giữ bản xem trước và đường dẫn khôi phục. Ở các lượt sau, 3 kết quả vừa đọc gần nhất vẫn giữ nguyên vẹn trong khi các kết quả dài cũ hơn sẽ chuyển thành dạng tham chiếu `[Earlier tool result saved at ...]`.

### Thử nghiệm 2: Lưu kết quả lớn ra đĩa

```text
Analyze the structure of web/src/data/generated/docs.json
and explain the main fields in one lesson record.
```

Khi file vượt quá hạn mức ngân sách của một lượt, tác vụ vẫn có thể hoàn thành và toàn bộ kết quả đầy đủ sẽ xuất hiện trong thư mục `.task_outputs/tool-results/`.

### Thử nghiệm 3: Kích hoạt tóm tắt tự động

```text
Compare s08_context_compact/code.py with s09_memory/code.py.
Explain how they manage current context and persistent memory.
```

Khi kết quả đọc file đẩy `estimate_chars(messages)` vượt quá 50000, terminal sẽ in `[auto compact]` kèm đường dẫn bản ghi lịch sử. Cuộc gọi tiếp theo sẽ tiếp tục từ bản tóm tắt `[Compacted]`.

Hãy kiểm tra hai thư mục `.transcripts/` và `.task_outputs/tool-results/` để xem các bản lưu trữ lịch sử cùng những tệp kết quả lớn được lưu lại.


## Tiếp Theo

Thu gọn ngữ cảnh giúp Agent tiếp tục hoàn thành các tác vụ dài trong một cửa sổ giới hạn. Tuy nhiên, những thông tin cần tồn tại qua các đợt thu gọn và xuyên suốt các phiên làm việc trong tương lai đòi hỏi một hệ thống bộ nhớ bền vững (persistent memory) chuyên biệt.

s09 Memory sẽ bổ sung cơ chế ghi nhớ, truy xuất và hợp nhất tri thức.


<!-- translation-sync: zh@v8, en@v8, ja@v8 -->
