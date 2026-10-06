# s03: Lập Kế Hoạch Với TodoWrite (TodoWrite)

`s01 > s02 > [ s03 ] > s04 > s05 > s06 | s07 > s08 > s09 > s10 > s11 > s12`

> *"An agent without a plan drifts"* -- lên danh sách các bước trước, rồi thực thi theo trình tự.
>
> **Tầng Harness**: Lập kế hoạch (Planning) -- giữ mô hình đi đúng hướng mà không cần ép cứng kịch bản đường đi.

## Vấn đề

Khi xử lý các tác vụ gồm nhiều bước phức tạp, mô hình rất dễ mất phương hướng. Nó có thể lặp lại công việc đã làm, bỏ sót các bước quan trọng hoặc đi chệch khỏi yêu cầu ban đầu. Cuộc hội thoại càng dài thì tình trạng này càng tệ hơn -- prompt hệ thống dần bị lu mờ khi kết quả trả về từ các công cụ lấp đầy ngữ cảnh (context). Một đợt tái cấu trúc code (refactor) gồm 10 bước có thể hoàn thành trôi chảy ở bước 1-3, nhưng sau đó mô hình bắt đầu tự biên tự diễn vì đã quên mất các bước từ 4 đến 10.

## Giải pháp

```
+--------+      +-------+      +---------+
|  User  | ---> |  LLM  | ---> | Tools   |
| prompt |      |       |      | + todo  |
+--------+      +---+---+      +----+----+
                    ^                |
                    |   tool_result  |
                    +----------------+
                          |
              +-----------+-----------+
              | TodoManager state     |
              | [ ] task A            |
              | [>] task B  <- doing  |
              | [x] task C            |
              +-----------------------+
                          |
              if rounds_since_todo >= 3:
                inject <reminder> into tool_result
```

## Cách thức hoạt động

1. `TodoManager` lưu trữ các đầu việc kèm theo trạng thái. Chỉ cho phép duy nhất một việc ở trạng thái `in_progress` (đang thực hiện) tại một thời điểm.

```python
class TodoManager:
    def update(self, items: list) -> str:
        validated, in_progress_count = [], 0
        for item in items:
            status = item.get("status", "pending")
            if status == "in_progress":
                in_progress_count += 1
            validated.append({"id": item["id"], "text": item["text"],
                              "status": status})
        if in_progress_count > 1:
            raise ValueError("Only one task can be in_progress")
        self.items = validated
        return self.render()
```

2. Công cụ `todo` được đưa vào bảng điều phối giống như bất kỳ công cụ nào khác.

```python
TOOL_HANDLERS = {
    # ...base tools...
    "todo": lambda **kw: TODO.update(kw["items"]),
}
```

3. Một cơ chế nhắc nhở (nag reminder) sẽ chèn lời nhắc nếu mô hình trải qua từ 3 vòng tương tác trở lên mà không gọi công cụ `todo`.

```python
if rounds_since_todo >= 3 and messages:
    last = messages[-1]
    if last["role"] == "user" and isinstance(last.get("content"), list):
        last["content"].insert(0, {
            "type": "text",
            "text": "<reminder>Update your todos.</reminder>",
        })
```

Ràng buộc "chỉ một đầu việc in_progress tại một thời điểm" buộc mô hình phải tập trung xử lý tuần tự. Lời nhắc nhở tạo ra tính kỷ luật và trách nhiệm theo dõi tiến độ.

## Những điểm thay đổi so với s02

| Thành phần       | Trước đây (s02)     | Sau khi cập nhật (s03)        |
|------------------|---------------------|-------------------------------|
| Công cụ          | 4                   | 5 (bổ sung thêm todo)         |
| Lập kế hoạch     | Không có            | TodoManager quản lý trạng thái|
| Nhắc nhở tiến độ | Không có            | `<reminder>` sau mỗi 3 vòng   |
| Vòng lặp Agent   | Điều phối đơn giản  | + bộ đếm `rounds_since_todo`  |

## Trải nghiệm thực tế

```sh
cd learn-claude-code
python agents/s03_todo_write.py
```

Hãy thử các prompt sau:

1. `Refactor the file hello.py: add type hints, docstrings, and a main guard`
2. `Create a Python package with __init__.py, utils.py, and tests/test_utils.py`
3. `Review all Python files and fix any style issues`
