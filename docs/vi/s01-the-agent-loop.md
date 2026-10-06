# s01: Vòng Lặp Agent (The Agent Loop)

`[ s01 ] > s02 > s03 > s04 > s05 > s06 | s07 > s08 > s09 > s10 > s11 > s12`

> *"One loop & Bash is all you need"* -- một công cụ + một vòng lặp = một agent.
>
> **Tầng Harness**: Vòng lặp -- sợi dây kết nối đầu tiên của mô hình với thế giới thực.

## Vấn đề

Một mô hình ngôn ngữ có thể tư duy suy luận về code, nhưng nó không thể *chạm* vào thế giới thực -- không thể đọc file, chạy kiểm thử hay kiểm tra lỗi. Nếu không có vòng lặp, mỗi lần gọi công cụ bạn đều phải tự sao chép-dán kết quả ngược trở lại bằng tay. Chính bạn đã trở thành vòng lặp đó.

## Giải pháp

```
+--------+      +-------+      +---------+
|  User  | ---> |  LLM  | ---> |  Tool   |
| prompt |      |       |      | execute |
+--------+      +---+---+      +----+----+
                    ^                |
                    |   tool_result  |
                    +----------------+
                    (loop until stop_reason != "tool_use")
```

Chỉ một điều kiện thoát duy nhất kiểm soát toàn bộ luồng. Vòng lặp tiếp tục chạy cho đến khi mô hình dừng gọi công cụ.

## Cách thức hoạt động

1. Prompt của người dùng trở thành tin nhắn đầu tiên.

```python
messages.append({"role": "user", "content": query})
```

2. Gửi các tin nhắn cùng với định nghĩa công cụ tới LLM.

```python
response = client.messages.create(
    model=MODEL, system=SYSTEM, messages=messages,
    tools=TOOLS, max_tokens=8000,
)
```

3. Thêm phản hồi của assistant vào danh sách. Kiểm tra `stop_reason` -- nếu mô hình không gọi công cụ, chúng ta hoàn tất.

```python
messages.append({"role": "assistant", "content": response.content})
if response.stop_reason != "tool_use":
    return
```

4. Thực thi từng lệnh gọi công cụ, thu thập kết quả, thêm vào dưới dạng tin nhắn của user. Lặp lại bước 2.

```python
results = []
for block in response.content:
    if block.type == "tool_use":
        output = run_bash(block.input["command"])
        results.append({
            "type": "tool_result",
            "tool_use_id": block.id,
            "content": output,
        })
messages.append({"role": "user", "content": results})
```

Ráp lại thành một hàm hoàn chỉnh:

```python
def agent_loop(query):
    messages = [{"role": "user", "content": query}]
    while True:
        response = client.messages.create(
            model=MODEL, system=SYSTEM, messages=messages,
            tools=TOOLS, max_tokens=8000,
        )
        messages.append({"role": "assistant", "content": response.content})

        if response.stop_reason != "tool_use":
            return

        results = []
        for block in response.content:
            if block.type == "tool_use":
                output = run_bash(block.input["command"])
                results.append({
                    "type": "tool_result",
                    "tool_use_id": block.id,
                    "content": output,
                })
        messages.append({"role": "user", "content": results})
```

Đó là toàn bộ agent trong chưa đầy 30 dòng code. Mọi bài viết khác trong khóa học này đều được xếp tầng lên trên nền tảng này -- mà không làm thay đổi vòng lặp.

## Những điểm thay đổi

| Thành phần    | Trước đây  | Sau khi có vòng lặp            |
|---------------|------------|--------------------------------|
| Vòng lặp Agent| (không có) | `while True` + stop_reason     |
| Công cụ       | (không có) | `bash` (một công cụ duy nhất)  |
| Tin nhắn      | (không có) | Danh sách tích lũy             |
| Luồng điều khiển | (không có) | `stop_reason != "tool_use"` |

## Trải nghiệm thực tế

```sh
cd learn-claude-code
python agents/s01_agent_loop.py
```

Hãy thử các prompt sau (prompt tiếng Anh thường hoạt động tối ưu với LLM, bạn cũng có thể thử bằng tiếng Việt):

1. `Create a file called hello.py that prints "Hello, World!"`
2. `List all Python files in this directory`
3. `What is the current git branch?`
4. `Create a directory called test_output and write 3 files in it`
