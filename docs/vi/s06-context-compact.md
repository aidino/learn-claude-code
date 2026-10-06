# s06: Thu Gọn Ngữ Cảnh (Context Compact)

`s01 > s02 > s03 > s04 > s05 > [ s06 ] | s07 > s08 > s09 > s10 > s11 > s12`

> *"Context will fill up; you need a way to make room"* -- cửa sổ ngữ cảnh luôn có giới hạn; bạn cần một chiến lược nén đa tầng để duy trì phiên làm việc vô hạn.
>
> **Tầng Harness**: Thu gọn ngữ cảnh (Compression) -- dọn sạch bộ nhớ cho các phiên làm việc kéo dài bất tận.

## Vấn đề

Cửa sổ ngữ cảnh (context window) luôn có giới hạn hữu hạn. Một lệnh `read_file` trên một file dài 1.000 dòng có thể tiêu tốn khoảng 4.000 token. Sau khi đọc 30 file và thực thi 20 lệnh bash, bạn sẽ nhanh chóng chạm ngưỡng hơn 100.000 token. Agent không thể làm việc trên các codebase lớn nếu thiếu cơ chế thu gọn ngữ cảnh.

## Giải pháp

Chiến lược 3 tầng thu gọn, tăng dần theo mức độ can thiệp:

```
Mỗi lượt (turn):
+------------------+
| Kết quả công cụ  |
+------------------+
        |
        v
[Tầng 1: micro_compact]            (chạy ngầm, mỗi lượt tương tác)
  Thay thế tool_result cũ > 3 lượt
  bằng "[Previous: used {tool_name}]"
        |
        v
[Kiểm tra: tokens > 50000?]
   |               |
  không            có
   |               |
   v               v
tiếp tục    [Tầng 2: auto_compact]
              Lưu toàn bộ biên bản (transcript) vào .transcripts/
              LLM tóm tắt lại cuộc hội thoại.
              Thay thế toàn bộ tin nhắn bằng [summary].
                    |
                    v
            [Tầng 3: công cụ compact]
              Mô hình chủ động gọi tool compact.
              Cùng quy trình tóm tắt như auto_compact.
```

## Cách thức hoạt động

1. **Tầng 1 -- micro_compact (Thu gọn vi mô)**: Trước mỗi lần gọi LLM, thay thế các kết quả gọi công cụ cũ bằng văn bản giữ chỗ (placeholder).

```python
def micro_compact(messages: list) -> list:
    tool_results = []
    for i, msg in enumerate(messages):
        if msg["role"] == "user" and isinstance(msg.get("content"), list):
            for j, part in enumerate(msg["content"]):
                if isinstance(part, dict) and part.get("type") == "tool_result":
                    tool_results.append((i, j, part))
    if len(tool_results) <= KEEP_RECENT:
        return messages
    for _, _, part in tool_results[:-KEEP_RECENT]:
        if len(part.get("content", "")) > 100:
            part["content"] = f"[Previous: used {tool_name}]"
    return messages
```

2. **Tầng 2 -- auto_compact (Tự động thu gọn)**: Khi số lượng token vượt quá ngưỡng định sẵn, lưu toàn bộ biên bản cuộc hội thoại ra ổ đĩa, sau đó yêu cầu LLM tóm tắt lại.

```python
def auto_compact(messages: list) -> list:
    # Lưu biên bản để có thể phục hồi khi cần
    transcript_path = TRANSCRIPT_DIR / f"transcript_{int(time.time())}.jsonl"
    with open(transcript_path, "w") as f:
        for msg in messages:
            f.write(json.dumps(msg, default=str) + "\n")
    # LLM thực hiện tóm tắt
    response = client.messages.create(
        model=MODEL,
        messages=[{"role": "user", "content":
            "Summarize this conversation for continuity..."
            + json.dumps(messages, default=str)[:80000]}],
        max_tokens=2000,
    )
    return [
        {"role": "user", "content": f"[Compressed]\n\n{response.content[0].text}"},
    ]
```

3. **Tầng 3 -- compact thủ công qua công cụ**: Công cụ `compact` kích hoạt quy trình tóm tắt tương tự theo yêu cầu chủ động của agent.

4. Vòng lặp tích hợp cả 3 tầng:

```python
def agent_loop(messages: list):
    while True:
        micro_compact(messages)                        # Tầng 1
        if estimate_tokens(messages) > THRESHOLD:
            messages[:] = auto_compact(messages)       # Tầng 2
        response = client.messages.create(...)
        # ... thực thi công cụ ...
        if manual_compact:
            messages[:] = auto_compact(messages)       # Tầng 3
```

Biên bản lưu trữ (transcripts) bảo toàn trọn vẹn toàn bộ lịch sử trên ổ đĩa. Không có dữ liệu nào thực sự bị mất đi -- chúng chỉ được đưa ra khỏi vùng ngữ cảnh đang hoạt động.

## Những điểm thay đổi so với s05

| Thành phần       | Trước đây (s05)   | Sau khi cập nhật (s06)       |
|------------------|-------------------|------------------------------|
| Công cụ          | 5                 | 5 (cơ sở + compact)          |
| Quản lý ngữ cảnh | Không có          | Thu gọn ba tầng              |
| Micro-compact    | Không có          | Thay thế kết quả cũ -> placeholder |
| Auto-compact     | Không có          | Kích hoạt theo ngưỡng token  |
| Lưu trữ biên bản | Không có          | Ghi file vào .transcripts/   |

## Trải nghiệm thực tế

```sh
cd learn-claude-code
python agents/s06_context_compact.py
```

Hãy thử các prompt sau:

1. `Read every Python file in the agents/ directory one by one` (quan sát micro-compact thay thế kết quả cũ)
2. `Keep reading files until compression triggers automatically`
3. `Use the compact tool to manually compress the conversation`
