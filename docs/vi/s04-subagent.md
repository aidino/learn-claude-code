# s04: Subagent (Agent Con)

`s01 > s02 > s03 > [ s04 ] > s05 > s06 | s07 > s08 > s09 > s10 > s11 > s12`

> *"Break big tasks down; each subtask gets a clean context"* -- chia nhỏ tác vụ lớn; mỗi tác vụ con nhận một ngữ cảnh (context) hoàn toàn sạch. Subagent sử dụng mảng messages[] độc lập, giữ cho cuộc hội thoại chính không bị ô nhiễm.
>
> **Tầng Harness**: Cô lập ngữ cảnh (Context isolation) -- bảo vệ sự minh mẫn trong suy luận của mô hình.

## Vấn đề

Khi agent làm việc, mảng tin nhắn (messages) của nó phình to liên tục. Mọi lần đọc file, mọi kết quả stdout của bash đều tồn tại vĩnh viễn trong ngữ cảnh. Một câu hỏi như "Dự án này sử dụng framework kiểm thử nào?" có thể đòi hỏi đọc tới 5 file cấu hình khác nhau, nhưng agent cha (parent) thực chất chỉ cần câu trả lời ngắn gọn: "pytest."

## Giải pháp

```
Agent cha (Parent agent)            Subagent (Agent con)
+------------------+             +------------------+
| messages=[...]   |             | messages=[]      | <-- ngữ cảnh mới tinh
|                  |  điều phối  |                  |
| tool: task       | ----------> | while tool_use:  |
|   prompt="..."   | (dispatch)  |   gọi công cụ    |
|                  |             |   nối kết quả    |
|                  |  tóm tắt    |                  |
|   result = "..." | <---------- | trả về text cuối |
+------------------+  (summary)  +------------------+

Ngữ cảnh của agent cha luôn sạch sẽ. Ngữ cảnh của subagent được giải phóng sau khi xong.
```

## Cách thức hoạt động

1. Agent cha nhận thêm công cụ `task`. Agent con nhận tất cả các công cụ cơ sở ngoại trừ `task` (tránh việc sinh subagent đệ quy vô hạn).

```python
PARENT_TOOLS = CHILD_TOOLS + [
    {"name": "task",
     "description": "Spawn a subagent with fresh context.",
     "input_schema": {
         "type": "object",
         "properties": {"prompt": {"type": "string"}},
         "required": ["prompt"],
     }},
]
```

2. Subagent bắt đầu với `messages=[]` và chạy vòng lặp riêng biệt của chính nó. Chỉ có văn bản kết quả cuối cùng được trả về cho agent cha.

```python
def run_subagent(prompt: str) -> str:
    sub_messages = [{"role": "user", "content": prompt}]
    for _ in range(30):  # giới hạn an toàn
        response = client.messages.create(
            model=MODEL, system=SUBAGENT_SYSTEM,
            messages=sub_messages,
            tools=CHILD_TOOLS, max_tokens=8000,
        )
        sub_messages.append({"role": "assistant",
                             "content": response.content})
        if response.stop_reason != "tool_use":
            break
        results = []
        for block in response.content:
            if block.type == "tool_use":
                handler = TOOL_HANDLERS.get(block.name)
                output = handler(**block.input)
                results.append({"type": "tool_result",
                    "tool_use_id": block.id,
                    "content": str(output)[:50000]})
        sub_messages.append({"role": "user", "content": results})
    return "".join(
        b.text for b in response.content if hasattr(b, "text")
    ) or "(no summary)"
```

Toàn bộ lịch sử tin nhắn của agent con (có thể chứa hơn 30 lượt gọi công cụ) sẽ được hủy bỏ. Agent cha nhận lại một đoạn văn bản tóm tắt ngắn gọn như một `tool_result` bình thường.

## Những điểm thay đổi so với s03

| Thành phần       | Trước đây (s03)   | Sau khi cập nhật (s04)           |
|------------------|-------------------|----------------------------------|
| Công cụ          | 5                 | 5 (cơ sở) + task (cho agent cha) |
| Ngữ cảnh         | Dùng chung duy nhất| Cô lập ngữ cảnh cha và con      |
| Subagent         | Không có          | Hàm `run_subagent()`             |
| Giá trị trả về   | Không áp dụng     | Chỉ trả về văn bản tóm tắt       |

## Trải nghiệm thực tế

```sh
cd learn-claude-code
python agents/s04_subagent.py
```

Hãy thử các prompt sau:

1. `Use a subtask to find what testing framework this project uses`
2. `Delegate: read all .py files and summarize what each one does`
3. `Use a task to create a new module, then verify it from here`
