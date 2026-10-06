# s05: TodoWrite — Agent Không Có Kế Hoạch Sẽ Lạc Lối

[English](README.md) · [中文](README.zh.md) · [日本語](README.ja.md) · [Tiếng Việt](README.vi.md)

s01 → s02 → s03 → s04 → `s05` → [s06](../s06_subagent/) → s07 → ... → s16 → s17

> *"An agent without a plan goes wherever the wind blows"* — Liệt kê các bước trước, sau đó mới thực thi. Các tác vụ phức tạp sẽ ít bị bỏ sót bước hơn.
>
> **Lớp Harness**: Lập kế hoạch (Planning) — Giúp Agent suy nghĩ thấu đáo trước khi hành động.

---

## Vấn đề Đặt ra

Giao cho Agent một tác vụ phức tạp: "Đổi tên tất cả các file Python sang chuẩn snake_case, chạy kiểm thử, và sửa các lỗi phát sinh."

Agent bắt đầu làm, đổi tên được 3 file, chạy thử test, thấy 2 ca kiểm thử thất bại, liền bắt tay vào sửa lỗi. Trong quá trình sửa lỗi, nó quên mất mục tiêu ban đầu là "đổi tên sang snake_case", các lỗi kiểm thử đã chiếm trọn toàn bộ sự chú ý của nó.

Cuộc trò chuyện càng kéo dài thì tình trạng càng tồi tệ: kết quả thực thi công cụ liên tục lấp đầy ngữ cảnh, làm loãng dần tầm ảnh hưởng của system prompt. Một đợt tái cấu trúc gồm 10 bước: sau khi xong bước 1-3, Agent bắt đầu tùy cơ ứng biến tự phát vì các bước từ 4-10 đã bị đẩy ra ngoài vùng chú ý của nó.

---

## Giải pháp

![Todo Overview](images/todo-overview.en.svg)

S05 duy trì toàn bộ cơ chế điều phối công cụ, phân quyền và các hook từ S04, sau đó bổ sung thêm công cụ `todo_write` cùng một bộ đếm nhắc nhở. `todo_write` chỉ cập nhật trạng thái kế hoạch; các công cụ hiện có vẫn chịu trách nhiệm thực thi công việc thực tế.

Công cụ mới sử dụng chung luồng điều phối `TOOL_HANDLERS[block.name]`. Sau ba vòng gọi công cụ liên tiếp mà không sử dụng `todo_write`, harness sẽ tự động chèn thêm một lời nhắc vào kết quả công cụ của vòng đó.

---

## Cơ chế Hoạt động

**TodoManager** quản lý danh sách công việc trong bộ nhớ, xác thực các bản cập nhật và kết xuất (render) trạng thái trả về cho mô hình. Hàm `run_todo_write` cũng đồng thời in trạng thái đó ra terminal:

```python
class TodoManager:
    def __init__(self):
        self.items = []

    def update(self, todos: list | str) -> str:
        # Phân tích và xác thực trước khi thay thế danh sách hiện tại.
        validated = []
        ...
        self.items = validated
        return self.render()

    def render(self) -> str:
        # [ ] pending, [>] in progress, [x] completed
        ...


TODO = TodoManager()

def run_todo_write(todos: list | str) -> str:
    output = TODO.update(todos)
    print(output)
    return output
```

Mỗi lần cập nhật có tối đa 20 mục công việc, mỗi mục bắt buộc có `content` không được rỗng, và chỉ được phép có duy nhất một mục ở trạng thái `in_progress`. Chuỗi đầu vào có thể là JSON hoặc biểu diễn danh sách Python mà không cần dùng hàm `eval`.

Định nghĩa công cụ được thêm vào danh sách cùng 5 công cụ trước trong bảng điều phối:

```python
TOOLS = [
    {"name": "bash",       ...},
    {"name": "read_file",  ...},
    {"name": "write_file", ...},
    {"name": "edit_file",  ...},
    {"name": "glob",       ...},
    # s05: mục mới
    {"name": "todo_write", "description": "Create and manage a task list ...",
     "input_schema": {
         "type": "object",
         "properties": {
             "todos": {
                 "type": "array",
                 "items": {
                     "type": "object",
                     "properties": {
                         "content": {"type": "string"},
                         "status": {"type": "string", "enum": ["pending", "in_progress", "completed"]},
                     },
                 },
             },
         },
     },
    },
]

TOOL_HANDLERS["todo_write"] = run_todo_write
```

**Lời nhắc (Reminder)**: sau ba vòng gọi công cụ mà không gọi `todo_write`, lời nhắc sẽ được nối vào kết quả của vòng thứ ba và bộ đếm được đặt lại về 0:

```python
rounds_since_todo = 0 if used_todo else rounds_since_todo + 1
if rounds_since_todo >= 3:
    results.append({
        "type": "text",
        "text": "<reminder>Update your todos.</reminder>",
    })
    rounds_since_todo = 0
```

Quy trình điển hình khi Agent nhận một tác vụ: đầu tiên gọi `todo_write` để liệt kê tất cả các bước (tất cả đều ở trạng thái `pending`) → chọn một bước, chuyển sang `in_progress` → hoàn thành xong, chuyển sang `completed` → xem bước `pending` tiếp theo → tiếp tục thực hiện.

**Điểm cốt lõi**: `todo_write` không mang lại cho Agent thêm **khả năng thực thi** nào. Thứ mà nó bổ sung chính là **khả năng lập kế hoạch**.

---

## Thay Đổi So Với s04

| Thành phần | Trước (s04) | Sau (s05) |
|------------|-------------|-----------|
| Số lượng công cụ | 5 (bash, read, write, edit, glob) | 6 (+todo_write) |
| Lập kế hoạch | Không có | Danh sách TODO có lưu trạng thái + bộ nhắc nhở |
| SYSTEM prompt | Prompt chung | Bổ sung hướng dẫn "lập kế hoạch trước khi thực thi" |
| Vòng lặp | Điều phối công cụ và hooks | Cùng luồng điều phối, cộng thêm đếm rounds_since_todo và tiêm lời nhắc |

---

## Thử Nghiệm

```sh
cd learn-claude-code
python s05_todo_write/code.py
```

Hãy thử các câu lệnh mẫu sau:

1. `Refactor s05_todo_write/example/hello.py: add type hints, docstrings, and a main guard` (Agent sẽ liệt kê 3 bước trước, sau đó mới thực thi)
2. `Create a Python package under s05_todo_write/example/demo_pkg with __init__.py, utils.py, and tests/test_utils.py`
3. `Review Python files under s05_todo_write/example and fix any style issues`

Điểm cần quan sát: Lệnh gọi công cụ đầu tiên có phải là `todo_write` không? Có bao nhiêu bước TODO được liệt kê? Trạng thái công việc có chuyển đổi tuần tự từ `pending` sang `in_progress` / `completed` trong suốt quá trình chạy không?

---

## Tiếp Theo

Bây giờ Agent đã biết lập kế hoạch. Nhưng nếu một tác vụ quá lớn, chẳng hạn "tái cấu trúc toàn bộ module xác thực", thì chỉ riêng danh sách TODO là không đủ. Bản thân tác vụ đó chứa hàng tá công việc con mà nếu dồn hết vào một ngữ cảnh trò chuyện đơn lẻ thì context sẽ nhanh chóng bị quá tải.

→ s06 Subagent: Chia nhỏ tác vụ lớn thành các tác vụ con, mỗi tác vụ con do một Agent độc lập xử lý với ngữ cảnh sạch sẽ riêng biệt, không gây nhiễu lẫn nhau.


<!-- translation-sync: zh@v1, en@v1, ja@v1 -->
