# s11: Tác Vụ Chạy Ngầm — Thao Tác Chậm Chuyển Xuống Nền

[English](README.md) · [中文](README.zh.md) · [日本語](README.ja.md) · [Tiếng Việt](README.vi.md)

s01 → ... → s09 → s10 → `s11` → [s12](../s12_cron_scheduler/) → s13 → ... → s16 → s17

> *"Thao tác chậm chuyển xuống nền, Vòng lặp Agent tiếp tục chạy"* — Các luồng chạy ngầm thực thi lệnh, và các lượt tương tác sau sẽ thu thập kết quả đã hoàn thành.
>
> **Lớp Harness**: Chạy ngầm (Background) — Thực thi bất đồng bộ, không làm nghẽn vòng lặp chính.

---

## Vấn Đề Đặt Ra

Đọc một file hoặc chạy lệnh `git status` thường trả về rất nhanh, vì vậy việc thực thi đồng bộ hầu như không gây ra độ trễ đáng chú ý. Tuy nhiên, việc cài đặt các gói phụ thuộc, chạy toàn bộ bộ kiểm thử, hay build một dự án có thể mất vài phút. Cho đến khi câu lệnh trả về kết quả, Harness không thể xử lý lệnh gọi công cụ tiếp theo trong phản hồi hiện tại hoặc bắt đầu lượt tương tác mới với mô hình.

Nếu các công việc tiếp theo không phụ thuộc vào câu lệnh đó, việc chặn toàn bộ chu trình là không cần thiết. Ví dụ, sau khi bắt đầu chạy bộ kiểm thử đầy đủ, Agent có thể kiểm tra tài liệu hoặc sắp xếp các file khác trong khi các bài kiểm thử vẫn đang chạy.

S11 giải quyết vấn đề này bằng cách chạy các lệnh Bash tốn thời gian dưới nền, cho phép Vòng lặp Agent tiếp tục hoạt động và thu thập kết quả đã hoàn thành ở một lượt tương tác sau.

---

## Giải Pháp

![Background Tasks Overview](images/background-tasks-overview.en.svg)

Chương này chuyển các thao tác chậm sang các luồng chạy ngầm (background threads). Lệnh gọi công cụ hiện tại trước tiên sẽ trả về một `tool_result` giữ chỗ (placeholder), cho phép Vòng lặp Agent tiếp tục. Khi bắt đầu một lượt tương tác sau, các kết quả đã hoàn thành được thu thập và đưa vào cuộc hội thoại dưới dạng thông báo.

So sánh Đồng bộ và Chạy ngầm:

| | Đồng bộ (s04) | Chạy ngầm (s11) |
|---|---|---|
| Thao tác chậm | Lệnh gọi công cụ hiện tại bị nghẽn (block) | Luồng chạy ngầm thực thi |
| Vòng lặp Agent | Chờ câu lệnh trả về kết quả | Tiếp tục chạy ngay sau kết quả giữ chỗ |
| Kết quả | Trả về sau khi câu lệnh kết thúc | Trả về `bg_id` trước; thu thập kết quả ở lượt sau |
| Tiêu chí quyết định | — | Tham số `run_in_background` của công cụ bash |

---

## Cách Thức Hoạt Động

### should_run_background: Yêu Cầu Rõ Ràng

Mô hình yêu cầu thực thi dưới nền thông qua tham số `run_in_background` của công cụ bash. Chỉ các lệnh gọi bash có tham số này được đặt tường minh thành `true` mới đi vào nhánh này. Các lệnh gọi khác vẫn chạy đồng bộ như bình thường.

```python
def should_run_background(tool_name: str, tool_input: dict) -> bool:
    return (
        tool_name == "bash"
        and tool_input.get("run_in_background") is True
    )
```

Harness không còn phải đoán dựa trên các từ khóa như `install`, `build`, hay `test`. Chính lệnh gọi công cụ sẽ chủ động chọn chế độ thực thi một cách rõ ràng.

### BackgroundManager: Quản Lý Thực Thi Dưới Nền Và Vòng Đời

`BackgroundManager` quản lý trạng thái tác vụ và hàng đợi hoàn thành. Hàm `start()` đăng ký một tác vụ, khởi chạy một luồng daemon, và trả về `bg_id` ngay lập tức:

```python
class BackgroundManager:
    def __init__(self):
        self.tasks = {}
        self.results = {}
        self._ready = []
        self._lock = threading.Lock()

    def start(self, block) -> str:
        # Đăng ký tác vụ, sau đó chạy _run() trong một daemon thread.
        ...

    def _run(self, task_id: str, command: str):
        output, exit_code = _run_bash_process(command)
        status = "completed" if exit_code == 0 else "failed"
        with self._lock:
            self.tasks[task_id]["status"] = status
            self.results[task_id] = _format_bash_result(output, exit_code)
            self._ready.append(task_id)
```

Mã thoát (exit code) khác 0 hoặc ngoại lệ từ worker sẽ khiến trạng thái trở thành `failed`. Shell được khởi động trong nhóm tiến trình (process group) riêng của nó. Khi lệnh kết thúc, hết thời gian chờ (timeout), hoặc Agent thoát bình thường hay qua tín hiệu `SIGTERM`, môi trường chạy sẽ dừng nhóm tiến trình gốc đó. Đây là cơ chế dọn dẹp vòng đời chứ không phải là môi trường hộp cát (sandbox): một tiến trình tạo phiên mới vẫn có thể thoát khỏi nhóm.

### collect_background_results: Thu Thập Thông Báo

Khi bắt đầu một lượt tương tác sau, hàm `collect()` sẽ lấy các kết quả đã hoàn thành ra khỏi hàng đợi và định dạng chúng thành các thông điệp `<task_notification>`:

```python
def collect_background_results() -> list[str]:
    return BACKGROUND.collect()
```

Các thông báo không sử dụng lại `tool_use_id` ban đầu. Lệnh gọi công cụ ban đầu đã được phản hồi bằng một `tool_result` giữ chỗ; khi kết quả hoàn thành được thu thập, nó được thêm vào như một sự kiện độc lập theo định dạng `task_notification`. Một `tool_use` vẫn nhận chính xác một `tool_result`.

### Tích Hợp Vào Vòng Lặp

Trước mỗi lệnh gọi LLM, Vòng lặp Agent sẽ thu thập các kết quả chạy ngầm đã hoàn thành. `execute_tool()` vẫn chạy hook `PreToolUse` trên luồng chính trước khi chọn thực thi đồng bộ hay chạy ngầm:

```python
while True:
    inject_background_results(messages)
    response = client.messages.create(...)

def execute_tool(block) -> str:
    blocked = trigger_hooks("PreToolUse", block)
    if blocked is not None:
        return str(blocked)
    if should_run_background(block.name, block.input):
        task_id = start_background_task(block)
        output = f"[Background task {task_id} started]"
    else:
        output = call_tool(block)
    trigger_hooks("PostToolUse", block, output)
    return output
```

Các thao tác chậm trước tiên trả về một tool_result giữ chỗ kèm theo `bg_id`. Một tác vụ hoàn thành không tự đánh thức Agent; hàm `inject_background_results()` sẽ thu thập nó trong lần chạy tiếp theo của Vòng lặp Agent.

### Kết Hợp Toàn Bộ Quy Trình

```
Lượt 1:
  LLM → bash "npm install" (run_in_background=true)
  → start_background_task → bg_0001
  → tool_result: "[Background task bg_0001 started]..."
  → LLM: "Được rồi, tôi sẽ kiểm tra sau. Giờ tôi sẽ đọc file cấu hình."

Lượt 2:
  LLM → read_file "package.json" (nhanh, đồng bộ)
  → tool_result: nội dung file

Lượt 3:
  → thu thập bg_0001 dưới dạng <task_notification>
  → LLM thấy: file cấu hình + thông báo cài đặt gói trong cùng một tin nhắn
```

Trong khi `npm install` đang chạy dưới nền, Vòng lặp Agent vẫn tiếp tục xử lý lệnh `read_file`.

---

## Những Gì s11 Bổ Sung

| Thành phần | Nhân s04 | s11 |
|-----------|-------------|-------------|
| Mô hình thực thi | Hoàn toàn đồng bộ | Thao tác chậm chuyển sang luồng ngầm + chèn thông báo |
| Schema của bash | `command` | `command` + `run_in_background` |
| Các hàm mới | — | `should_run_background`, `start_background_task`, `collect_background_results`, `inject_background_results` |
| Các kiểu mới | — | `BackgroundManager` |
| Định dạng thông báo | — | `<task_notification>` (không tái sử dụng tool_use_id) |
| Hành vi vòng lặp | Công cụ chạy đồng bộ | Thực thi ngầm tường minh, thu thập kết quả hoàn thành ở lượt sau |
| Công cụ | 5 | 5 (thêm một tham số vào schema của bash) |

---

## Thử Nghiệm

```sh
cd learn-claude-code
python s11_background_tasks/code.py
```

Hãy thử các câu lệnh prompt sau:

1. `Run pip list in the background and find all Python files in this directory`
2. `Run npm install (use run_in_background) and while waiting, read package.json`
3. `Run a short sleep in the background, then list all Markdown files`

Những điểm cần quan sát: Sau khi thiết lập tường minh `run_in_background`, câu lệnh có được điều phối xuống nền không? `bg_id` có được trả về không? Kết quả hoàn thành có được thu thập dưới định dạng `<task_notification>` ở lượt sau không?

---

## Bước Tiếp Theo

Các tác vụ chạy ngầm đã giải quyết vấn đề "thao tác chậm không làm nghẽn". Nhưng nếu bạn muốn thực hiện điều gì đó theo lịch trình thì sao? Chẳng hạn như "chạy kiểm thử mỗi sáng lúc 9 giờ" hoặc "kiểm tra trạng thái máy chủ mỗi 5 phút một lần".

s12 Cron Scheduler → Trang bị cho agent một chiếc đồng hồ báo thức.

<!-- translation-sync: zh@v7, en@v7, ja@v7 -->
