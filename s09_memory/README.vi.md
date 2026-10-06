# s09: Bộ Nhớ — Giữ Lại Tri Thức Hữu Ích Qua Các Phiên Làm Việc

[English](README.md) · [中文](README.zh.md) · [日本語](README.ja.md) · [Tiếng Việt](README.vi.md)

s01 → ... → s07 → s08 → `s09` → [s10](../s10_task_system/) → s11 → ... → s16 → s17
> *"Lưu giữ thông tin mà các tác vụ sau này sẽ cần đến."* Lưu trữ file + chỉ mục + chọn lọc theo độ liên quan + gợi nhớ theo nhu cầu.
>
> **Lớp Harness**: Bộ nhớ lưu trữ tri thức có thể tái sử dụng bên ngoài hội thoại và gợi nhớ lại cho các tác vụ liên quan.

---

## Vấn Đề Đặt Ra

Một Agent khi bắt đầu một phiên làm việc mới sẽ không có lịch sử hội thoại trước đó trong `messages`. Một sở thích lập trình, một sự thật về dự án, hay một manh mối gỡ lỗi từ phiên trước đó vẫn có thể rất quan trọng. Nếu không có bộ lưu trữ bền vững (persistent storage), người dùng sẽ phải cung cấp lại từ đầu.

Lưu toàn bộ bản ghi hội thoại (transcript) có thể đóng vai trò như một kho lưu trữ, nhưng việc gửi toàn bộ bản ghi đó trong mỗi yêu cầu là điều không thể mở rộng. Cuộc hội thoại ngày càng dài thêm, thông tin hữu ích trở nên khó định vị, và các dữ kiện cũ có thể không còn đúng nữa. Hệ thống bộ nhớ phải quyết định điều gì đáng giữ lại qua các phiên và bản ghi nào thực sự thuộc về tác vụ hiện tại.

![Memory Overview](images/memory-overview.en.svg)

---

## Tại Sao Không Đưa Tất Cả Vào System Prompt?

Cách tiếp cận trực tiếp là ghi các sở thích và dữ kiện dự án vào một file, rồi đưa toàn bộ file đó vào system prompt. Cách này giúp Agent ghi nhớ thông tin, nhưng mỗi lệnh gọi LLM đều phải gửi lại toàn bộ nội dung đó. Khi kho lưu trữ phình to, ngày càng nhiều nội dung không liên quan sẽ tiêu tốn token đầu vào và chiếm dụng không gian ngữ cảnh.

Bài học s07 đã chỉ ra một mô hình đọc tốt hơn: duy trì một danh mục chỉ mục ngắn gọn và chỉ nạp toàn bộ nội dung khi cần thiết. Kỹ năng (skills) do con người viết và chỉ đọc. Ngược lại, bộ nhớ (memory) cho phép Agent tự trích xuất thông tin từ hội thoại và tái sử dụng trong các tác vụ sau này.

Do đó, chương này cần bốn phần: lưu trữ, gợi nhớ, trích xuất và hợp nhất.

![Memory Subsystems](images/memory-subsystems.en.svg)

---

## Lưu Trữ: Mỗi Bản Ghi Một File

Mỗi mục bộ nhớ là một file Markdown nằm trong thư mục `.memory/`. Phần YAML frontmatter lưu trữ `name`, `description`, và `type`:

```markdown
---
name: user-preference-tabs
description: User prefers tabs for indentation
type: user
---

User prefers using tabs, not spaces, for indentation.
```

Có bốn loại bộ nhớ:

| Loại | Nội dung lưu trữ | Ví dụ |
|------|------------------|-------|
| user | Sở thích lâu dài của người dùng | "Dùng tab để thụt lề" |
| feedback | Lời chỉ dẫn/góp ý vẫn còn giá trị | "Không mock cơ sở dữ liệu" |
| project | Dữ kiện ổn định của dự án | "Việc viết lại xác thực là do yêu cầu tuân thủ" |
| reference | Con trỏ bên ngoài hoặc manh mối tra cứu | "Vấn đề đường ống đang được theo dõi trong Linear INGEST" |

`MEMORY.md` là file chỉ mục, với mỗi dòng đại diện cho một file bộ nhớ. Sau khi ghi, hàm `rebuild_memory_index()` sẽ tạo lại chỉ mục từ các file:

```python
def write_memory_file(name, mem_type, description, body):
    path = MEMORY_DIR / f"{memory_slug(name)}.md"
    path.write_text(
        memory_document(name, mem_type, description, body), encoding="utf-8"
    )
    rebuild_memory_index()
    return path
```

File chỉ mục hỗ trợ việc chọn lọc trong khi toàn bộ nội dung chi tiết vẫn nằm trong từng file riêng lẻ.

---

## Gợi Nhớ: Chọn Lọc Trước, Sau Đó Mới Nạp Đầy Đủ

Khi bắt đầu một yêu cầu của người dùng, hàm `select_relevant_memories()` gửi văn bản gần đây của người dùng và danh mục bộ nhớ tới một lệnh gọi mô hình nhẹ. Lệnh này chọn ra tối đa năm bản ghi có liên quan:

```python
prompt = (
    "Select memory records that are relevant to the current user request. "
    "Return only a JSON array of catalog indices, such as [0, 2]. "
    "Return [] when none are relevant."
)
```

Nếu lệnh gọi mô hình hoặc việc phân tích JSON thất bại, mã nguồn sẽ chuyển về phương án dự phòng là khớp từ khóa. Chỉ sau khi đã chọn lọc, `load_memories()` mới đọc các file tương ứng, kèm theo giới hạn về tổng dung lượng văn bản gợi nhớ.

```python
relevant_memories = load_memories(messages)
system = build_system(relevant_memories)
```

`build_system()` nêu rõ rằng nội dung gợi nhớ là kiến thức nền tảng, không phải mệnh lệnh mới từ người dùng. Yêu cầu hiện tại sẽ được ưu tiên khi có xung đột với bộ nhớ. Điều này giúp Agent tận dụng thông tin cũ mà không để các bản ghi cũ tự ý phát lệnh thay cho người dùng.

---

## Trích Xuất: Lưu Thông Tin Có Thể Tái Sử Dụng Sau Mỗi Lượt

Người dùng không phải lúc nào cũng nói "hãy nhớ điều này". Sau khi Agent hoàn thành phản hồi hiện tại, `extract_memories()` sẽ kiểm tra cuộc hội thoại và chỉ giữ lại thông tin có khả năng hữu ích sau này:

```python
tool_calls = [
    block for block in response.content if block.type == "tool_use"
]
if not tool_calls:
    force = trigger_hooks("Stop", messages)
    if force:
        messages.append({"role": "user", "content": force})
        continue
    if extract_memories(messages):
        consolidate_memories()
    return
```

Mô hình trả về các ứng viên chứ không phải các bản ghi được tự động cho phép ghi vào đĩa. Mỗi ứng viên mang một `scope`: chỉ có `persistent` mới biểu thị thông tin nên được duy trì qua các phiên sau. Phạm vi `current_task` bao gồm các mệnh lệnh dùng một lần, đường dẫn tạm thời và các hạn chế tạm thời.

Hàm `should_store_memory()` thực hiện kiểm tra phê duyệt cuối cùng. Nó từ chối các ứng viên không đầy đủ, các cụm từ đề cập đến phiên hoặc tác vụ hiện tại, và các bản ghi trùng lặp với bản ghi đã có. Ví dụ, "không tạo file trong phiên này" chỉ ràng buộc công việc hiện tại; nó không được phép duy trì thành quy tắc trong phiên tiếp theo.

---

## Hợp Nhất: Gộp Các Bản Ghi Trùng Lặp Và Lỗi Thời

Khi các file bộ nhớ tích lũy nhiều dần, một số sẽ trở nên trùng lặp, mâu thuẫn hoặc lỗi thời. Bản triển khai giảng dạy gọi `consolidate_memories()` sau khi kho lưu trữ đạt mười bản ghi và yêu cầu mô hình cung cấp danh sách đã được dọn dẹp.

Mã nguồn phân tích cú pháp và xác thực danh sách mới trước khi thay thế các file cũ. Đầu tiên, nó sao lưu nhanh (snapshot) các bản ghi hiện tại; nếu việc xóa hoặc ghi file thất bại, nó sẽ khôi phục bản gốc và tạo lại chỉ mục:

```python
snapshot = {
    path.name: path.read_text(encoding="utf-8")
    for path in MEMORY_DIR.glob("*.md")
    if path.name != MEMORY_INDEX.name
}

try:
    for path in MEMORY_DIR.glob("*.md"):
        if path.name != MEMORY_INDEX.name:
            path.unlink()
    for record in consolidated:
        path = MEMORY_DIR / f"{memory_slug(record['name'])}.md"
        path.write_text(memory_document(
            record["name"], record["type"],
            record["description"], record["body"],
        ), encoding="utf-8")
    rebuild_memory_index()
except Exception:
    for path in MEMORY_DIR.glob("*.md"):
        if path.name != MEMORY_INDEX.name:
            path.unlink()
    for filename, content in snapshot.items():
        (MEMORY_DIR / filename).write_text(content, encoding="utf-8")
    rebuild_memory_index()
    raise
```

Khóa học sử dụng một ngưỡng số lượng đơn giản. Một ứng dụng thực tế cũng phải chọn một lịch trình phù hợp với khối lượng dữ liệu và ngăn các tiến trình đồng thời ghi đè vào cùng một kho lưu trữ.

---

## Mã Nguồn Bài Học Này

| Thành phần | Triển khai |
|------------|------------|
| Agent Loop | Quản lý messages, lệnh gọi công cụ, kết quả công cụ và các điểm kích hoạt hook |
| Công cụ cơ sở | `bash`, `read_file`, `write_file`, `edit_file`, `glob` |
| Lưu trữ | Chỉ mục `.memory/MEMORY.md` + các bản ghi `.memory/*.md` |
| Gợi nhớ | Lựa chọn theo danh mục + dự phòng từ khóa + giới hạn dung lượng phần thân |
| Ghi nhớ | Trích xuất cuối lượt + kiểm tra tính bền vững + lọc trùng lặp |
| Hợp nhất | Hợp nhất khi đạt ngưỡng; khôi phục file cũ nếu thay thế thất bại |

> **Ranh giới với s08:** s08 quản lý ngân sách ngữ cảnh của phiên làm việc đang hoạt động. s09 quản lý tri thức có thể tái sử dụng bên ngoài cuộc hội thoại. Bộ nhớ là cơ chế lưu trữ có chọn lọc, không phải bản sao lưu toàn vẹn lịch sử hội thoại, và nó không thay thế cho việc thu gọn ngữ cảnh.

---

## Thử Nghiệm

```sh
cd learn-claude-code
python s09_memory/code.py
```

1. Nhập `I prefer using tabs for indentation. Remember that.` Sau lượt đó, hãy kiểm tra thư mục `.memory/` xem có bản ghi mới và `MEMORY.md` có mục chỉ mục tương ứng hay không.
2. Nhập `q`, khởi động lại chương trình, và hỏi `What indentation style do I prefer?` Xác nhận rằng phiên mới có thể gợi nhớ lại sở thích này.
3. Lưu một sở thích khác không liên quan đến định dạng code, sau đó hỏi về thụt lề (indentation). Quan sát thấy yêu cầu hiện tại chỉ nạp các bản ghi có liên quan.
4. Nhập `Do not create files in this session.` Xác nhận rằng yêu cầu tạm thời này không trở thành quy tắc bền vững cho phiên tiếp theo.

Câu từ chính xác và số lượng trích xuất có thể khác nhau tùy thuộc vào mô hình. Hãy kiểm tra nội dung được ghi vào `.memory/` và liệu phiên sau có chỉ gợi nhớ thông tin liên quan hay không.

---

## Bước Tiếp Theo

Bộ nhớ lưu giữ thông tin qua các phiên làm việc, nhưng một tác vụ phức tạp cũng cần theo dõi trạng thái bền vững và các mối quan hệ phụ thuộc. Một TODO chỉ lưu trong hội thoại không thể duy trì tiến độ qua các lần khởi động lại tiến trình.

s10 Task System → Lưu trữ bền vững các tác vụ, trạng thái và phụ thuộc xuống đĩa.

<!-- translation-sync: zh@v3, en@v3, ja@v3 -->
