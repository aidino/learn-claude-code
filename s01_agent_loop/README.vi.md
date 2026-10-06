# s01: Agent Loop — Một Vòng Lặp Là Tất Cả Những Gì Bạn Cần

[English](README.md) · [中文](README.zh.md) · [日本語](README.ja.md) · [Tiếng Việt](README.vi.md)

`s01` → [s02](../s02_tool_use/) → s03 → s04 → ... → s16 → s17
> *"One loop & Bash is all you need"* — Một công cụ + một vòng lặp = một Agent.
>
> **Lớp Harness**: Vòng lặp (The Loop) — chiếc cầu nối đầu tiên giữa mô hình và thế giới thực.

---

## Vấn đề Đặt ra

Bạn yêu cầu mô hình: "Hãy liệt kê các file trong thư mục của tôi và chạy XXX.py."

Mô hình có thể sinh ra một lệnh bash, nhưng ngay khi trả về văn bản xong thì nó dừng lại — nó không thể tự mình thực thi lệnh đó, và cũng không thể tiếp tục suy luận dựa trên kết quả đầu ra.

Bạn có thể chạy lệnh đó thủ công, dán kết quả trở lại khung chat rồi bảo mô hình tiếp tục. Lệnh tiếp theo xuất hiện, bạn lại chạy thủ công, rồi lại dán vào.

Cứ mỗi lượt phản hồi, bạn chính là tầng trung gian xử lý thủ công. Tự động hóa quy trình đó chính là nội dung cốt lõi của bài học này.

---

## Giải pháp

![Agent Loop](images/agent-loop.en.svg)

Một vòng lặp `while True`: tiếp tục khi mô hình gọi công cụ, dừng lại khi nó không gọi nữa. Vòng lặp kiểm tra trực tiếp các khối nội dung (content blocks) trong phản hồi:

| Tín hiệu | Ý nghĩa | Hành động của Vòng lặp |
|----------|---------|------------------------|
| Chứa khối `tool_use` | Mô hình yêu cầu gọi công cụ | Thực thi → gửi trả kết quả → tiếp tục |
| Không chứa khối `tool_use` | Mô hình không gọi công cụ | Thoát vòng lặp |

---

## Cơ chế Hoạt động

Hãy chuyển hóa quy trình này thành mã nguồn từng bước một:

**Bước 1**: Bắt đầu với câu hỏi của người dùng làm tin nhắn đầu tiên.

```python
messages = [{"role": "user", "content": query}]
```

**Bước 2**: Gửi danh sách tin nhắn và định nghĩa công cụ (tool definitions) đến LLM.

```python
response = client.messages.create(
    model=MODEL, system=SYSTEM, messages=messages,
    tools=TOOLS, max_tokens=8000,
)
```

**Bước 3**: Nối thêm phản hồi của mô hình vào lịch sử và kiểm tra xem nó có gọi công cụ hay không. Không có công cụ nào được gọi → hoàn thành.

```python
messages.append({"role": "assistant", "content": response.content})
tool_calls = [
    block for block in response.content if block.type == "tool_use"
]
if not tool_calls:
    return
```

Chỉ những khối `tool_use` cụ thể mới bước vào giai đoạn thực thi, nhờ đó vòng lặp không bao giờ thêm vào một tin nhắn kết quả công cụ rỗng.

**Bước 4**: Thực thi công cụ mà mô hình yêu cầu và thu thập kết quả.

```python
results = []
for block in tool_calls:
    output = run_bash(block.input["command"])
    results.append({
        "type": "tool_result",
        "tool_use_id": block.id,
        "content": output,
    })
```

**Bước 5**: Thêm các kết quả công cụ thành một tin nhắn mới và quay lại Bước 2.

```python
messages.append({"role": "user", "content": results})
```

Ráp lại thành một hàm hoàn chỉnh:

```python
def agent_loop(messages):
    while True:
        response = client.messages.create(
            model=MODEL, system=SYSTEM, messages=messages,
            tools=TOOLS, max_tokens=8000,
        )
        messages.append({"role": "assistant", "content": response.content})

        tool_calls = [
            block for block in response.content if block.type == "tool_use"
        ]
        if not tool_calls:
            return

        results = []
        for block in tool_calls:
            output = run_bash(block.input["command"])
            results.append({
                "type": "tool_result",
                "tool_use_id": block.id,
                "content": output,
            })
        messages.append({"role": "user", "content": results})
```

Chỉ hơn 30 dòng code — đó chính là nhân (kernel) tối giản của một khung điều hành agent (agent harness) có thể chạy được. Nó không tự sinh ra trí thông minh, mà là khung runtime nhỏ nhất giúp mô hình liên tục hành động. Mô hình quyết định (có gọi công cụ hay không, gọi công cụ nào), harness thực thi (gọi công cụ và nối kết quả thành tin nhắn mới). Toàn bộ 16 bài học tiếp theo đều xây dựng các cơ chế bổ sung dựa trên vòng lặp này. Bản thân vòng lặp không bao giờ thay đổi.

---

## Thử nghiệm

> **Lưu ý an toàn**: Mã nguồn này trực tiếp thực thi các lệnh shell do mô hình sinh ra. Hãy chạy trong một thư mục kiểm thử tạm thời để tránh ảnh hưởng đến các file dự án của bạn. Bài s03 sẽ bổ sung cơ chế kiểm soát phân quyền.

**Cài đặt** (chạy lần đầu):

```sh
pip install -r requirements.txt
cp .env.example .env
# Chỉnh sửa file .env, điền ANTHROPIC_API_KEY và MODEL_ID
```

**Chạy**:

```sh
python s01_agent_loop/code.py
```

Hãy thử các câu lệnh mẫu sau:

1. `Create a file called hello.py that prints "Hello, World!"`
2. `List all Python files in this directory`
3. `What is the current git branch?`

Điểm cần quan sát: Khi nào mô hình gọi công cụ (vòng lặp tiếp tục), và khi nào nó không gọi (vòng lặp kết thúc)?

---

## Tiếp theo

Hiện tại mô hình chỉ có duy nhất công cụ bash — đọc file phải dùng `cat`, ghi file phải dùng `echo ... >`, tìm file phải dùng `find`. Cách này vừa rườm rà vừa rất dễ phát sinh lỗi.

→ s02 Tool Use: Điều gì sẽ xảy ra khi chúng ta trang bị cho mô hình 5 công cụ chuyên dụng thực thụ? Liệu mô hình có gọi nhiều công cụ cùng một lúc? Việc thực thi công cụ song song có xung đột lẫn nhau không?


<!-- translation-sync: zh@v2, en@v2, ja@v2 -->
