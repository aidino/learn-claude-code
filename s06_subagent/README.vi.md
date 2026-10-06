# s06: Subagent — Cung Cấp Ngữ Cảnh Độc Lập Cho Tác Vụ Con

[English](README.md) · [中文](README.zh.md) · [日本語](README.ja.md) · [Tiếng Việt](README.vi.md)

s01 → s02 → s03 → s04 → s05 → `s06` → [s07](../s07_skill_loading/) → s08 → ... → s16 → s17

> Một subagent bắt đầu với một danh sách `messages[]` hoàn toàn mới. Chỉ có văn bản kết quả cuối cùng được trả về cho agent cha; toàn bộ cuộc hội thoại trung gian thì không.
>
> **Lớp Harness**: Phân quyền ủy thác (Delegation) — Thực thi một tác vụ tập trung trong một ngữ cảnh hội thoại tách biệt.

---

## Vấn đề Đặt ra

Agent đang sửa một lỗi (bug). Nó phải đọc rất nhiều file để truy vết chuỗi gọi hàm (call chain), và mọi lệnh gọi công cụ cùng kết quả trả về đều lưu lại trong `messages[]` của agent cha. Một khi chuỗi gọi hàm đã được làm sáng tỏ, phần lớn những chi tiết trung gian đó không còn cần thiết nữa, nhưng chúng vẫn tiếp tục chiếm dụng không gian ngữ cảnh.

---

## Giải pháp

![Subagent Overview](images/subagent-overview.en.svg)

Việc gọi công cụ `task` sẽ đồng bộ kích hoạt một vòng lặp agent lồng nhau với một mảng `messages[]` hoàn toàn mới. Khi vòng lặp con kết thúc, văn bản kết luận cuối cùng của nó sẽ trở thành kết quả công cụ trong cuộc trò chuyện của agent cha.

Đây là sự cô lập về mặt hội thoại (message isolation), chứ không phải cô lập tiến trình hay cô lập hệ thống file. Agent cha và subagent cùng chạy trong một tiến trình Python và chia sẻ chung `WORKDIR`, do đó các thao tác ghi file và câu lệnh shell vẫn tác động trực tiếp lên cùng một không gian làm việc. Subagent sở hữu 5 công cụ cơ bản nhưng không có công cụ `task`, và các lệnh gọi công cụ của nó vẫn tuân thủ cùng các hook phân quyền và vòng đời như agent cha.

---

## Cơ chế Hoạt động

Hàm **run_subagent** khởi tạo danh sách tin nhắn mới, chạy vòng lặp lồng nhau và trả về văn bản kết quả cuối cùng:

```python
SUB_TOOLS = list(BASE_TOOLS)  # không chứa công cụ task

def run_subagent(prompt: str) -> str:
    messages = [{"role": "user", "content": prompt}]

    for _ in range(30):
        response = client.messages.create(
            model=MODEL, system=SUB_SYSTEM,
            messages=messages, tools=SUB_TOOLS, max_tokens=8000,
        )
        messages.append({"role": "assistant", "content": response.content})
        tool_calls = [
            block for block in response.content if block.type == "tool_use"
        ]
        if not tool_calls:
            return extract_text(response.content) or "(no summary)"

        results = []
        for block in tool_calls:
            output = execute_tool(block, SUB_HANDLERS)
            results.append({... "content": output})
        messages.append({"role": "user", "content": results})

    return "Subagent stopped after 30 turns without a final answer."
```

Agent chính gọi subagent tương tự như bất kỳ công cụ nào khác:

```python
TASK_TOOL = {
    "name": "task",
    "description": "Run a subagent with fresh conversation context and return its final text.",
    "input_schema": {
        "type": "object",
        "properties": {"prompt": {"type": "string"}},
        "required": ["prompt"],
    },
}

TOOLS = [*BASE_TOOLS, TASK_TOOL]
TOOL_HANDLERS = {**BASE_HANDLERS, "task": run_subagent}
```

Ranh giới thiết kế được xác định như sau:

| Quyết định | Lựa chọn | Lý do |
|------------|----------|-------|
| Cuộc hội thoại | `messages[]` hoàn toàn mới | Lịch sử của agent cha không bị sao chép vào subagent |
| Thực thi | Cùng tiến trình và `WORKDIR` | Các thay đổi trên hệ thống file hiển thị cho cả hai vòng lặp |
| Giá trị trả về | Chỉ văn bản cuối cùng | Các lệnh gọi công cụ và kết quả của con không bị sao chép vào tin nhắn cha |
| Độ sâu ủy thác | Không có `task` trong `SUB_TOOLS` | Bài học này chỉ cho phép một cấp độ ủy thác duy nhất |
| Chính sách công cụ | Dùng chung Hooks | Agent cha và subagent áp dụng cùng các bước kiểm tra phân quyền |

Agent cha điều phối `task` qua cùng một bảng ánh xạ handler như các công cụ khác. Subagent sử dụng `SUB_SYSTEM`, `SUB_TOOLS`, và danh sách `messages` cục bộ của riêng nó.

---

## Thử Nghiệm

```sh
cd learn-claude-code
python s06_subagent/code.py
```

Hãy thử các câu lệnh mẫu sau:

1. `Use a subtask to find what testing framework this project uses` (sub-Agent đọc các file, Agent chính chỉ nhận về kết luận ngắn gọn)
2. `Delegate: read all .py files in agents/ and summarize what each one does`
3. `Use a task to create s06_subagent/example/string_tools.py with a slugify(text: str) function, then verify it from the parent agent`

Điểm cần quan sát: Dòng thông báo `[Subagent started]` / `[Subagent done]` có xuất hiện không? Các lệnh gọi công cụ của subagent có được in ra dưới dạng `[sub] ...` không? Agent cha có tiếp tục công việc chỉ với văn bản kết luận do `task` trả về không?

---

## Tiếp Theo

Giờ đây Agent đã biết chia nhỏ tác vụ. Nhưng các tác vụ khác nhau lại đòi hỏi những tri thức chuyên biệt khác nhau: sửa component giao diện cần nắm quy ước React, viết SQL cần schema bảng dữ liệu. Nhồi nhét toàn bộ tri thức này vào system prompt sẽ lập tức làm bùng nổ ngữ cảnh.

→ s07 Skill Loading: Nạp kỹ năng theo yêu cầu thay vì chất đống tài liệu vào system prompt. Chỉ nạp đúng những gì cần thiết khi phát sinh nhu cầu, tự nhiên như việc đọc một file tài liệu.


<!-- translation-sync: zh@v2, en@v2, ja@v2 -->
