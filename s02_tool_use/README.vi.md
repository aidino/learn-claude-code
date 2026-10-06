# s02: Tool Use — Thêm Một Công Cụ, Thêm Đúng Một Dòng

[English](README.md) · [中文](README.zh.md) · [日本語](README.ja.md) · [Tiếng Việt](README.vi.md)

s01 → `s02` → [s03](../s03_permission/) → s04 → ... → s16 → s17
> *"Add a tool, add just one handler"* — Vòng lặp vẫn giữ nguyên. Chỉ cần đăng ký công cụ mới vào bảng điều phối (dispatch map) là hoàn thành.
>
> **Lớp Harness**: Điều phối công cụ (Tool Dispatch) — Mở rộng khả năng tác động của mô hình.

---

## Chỉ Có Một Công Cụ: Bash

Agent ở bài s01 chỉ có duy nhất một công cụ: bash. Để đọc một file, nó dùng `cat`; để ghi file, nó dùng `echo "..." > file.py`; để sửa file, nó dùng `sed`.

Mô hình nghĩ trong đầu là "đọc file này", nhưng lại phải diễn giải thành câu lệnh `cat path/to/file`. Tầng phiên dịch thừa thãi này vừa lãng phí token vừa làm tăng nguy cơ xảy ra lỗi.

---

## Tổng quan: Điều Phối Công Cụ (Tool Dispatch)

![Tool Dispatch](images/tool-dispatch.en.svg)

Vòng lặp từ s01 được giữ nguyên hoàn toàn (từ lệnh gọi LLM, kiểm tra khối `tool_use`, đến việc nối thêm tin nhắn — không đổi một chữ nào). Thay đổi duy nhất nằm ở đúng một dòng thực thi công cụ: thay thế hàm `run_bash()` cố định bằng cơ chế tra cứu điều phối `TOOL_HANDLERS[block.name]()`.

Thêm một công cụ vào Agent chỉ cần đúng hai bước:

1. **Định nghĩa công cụ**: Thêm một phần tử vào mảng `TOOLS`
2. **Đăng ký hàm xử lý**: Thêm một ánh xạ vào từ điển `TOOL_HANDLERS`

---

## Từ 1 Công Cụ Lên 5 Công Cụ

Ở s01 ta chỉ có bash:

```python
TOOLS = [{"name": "bash", ...}]

def run_bash(command): ...
```

Sang s02 mở rộng thành 5 công cụ, mỗi công cụ được định nghĩa độc lập:

```python
TOOLS = [
    {"name": "bash",       "description": "Run a shell command.", ...},
    {"name": "read_file",  "description": "Read file contents.",  ...},
    {"name": "write_file", "description": "Write content to file.", ...},
    {"name": "edit_file",  "description": "Replace text in file once.", ...},
    {"name": "glob",       "description": "Find files by pattern.", ...},
]
```

Mỗi công cụ có một hàm triển khai riêng biệt:

```python
def run_read(path, limit=None):
    lines = safe_path(path).read_text(encoding="utf-8").splitlines()
    if limit:
        lines = lines[:limit]
    return "\n".join(lines)

def run_write(path, content):
    safe_path(path).write_text(content, encoding="utf-8")
    return f"Wrote {len(content)} bytes to {path}"

def run_edit(path, old_text, new_text):
    text = safe_path(path).read_text(encoding="utf-8")
    if old_text not in text:
        return "Error: text not found"
    safe_path(path).write_text(text.replace(old_text, new_text, 1), encoding="utf-8")
    return f"Edited {path}"

def run_glob(pattern):
    import glob as g
    matches = sorted(set(g.glob(
        pattern, root_dir=WORKDIR, recursive=True)))
    shown = matches[:200]
    if len(matches) > 200:
        shown.append("... (more matches omitted; narrow the pattern)")
    return "\n".join(shown)
```

---

## Điều Phối Công Cụ (Tool Dispatch)

```python
TOOL_HANDLERS = {
    "bash":       run_bash,
    "read_file":  run_read,
    "write_file": run_write,
    "edit_file":  run_edit,
    "glob":       run_glob,
}

# Chỉ một dòng thay đổi trong vòng lặp — từ việc gọi cố định run_bash sang tra cứu điều phối:
for block in tool_calls:
    handler = TOOL_HANDLERS[block.name]    # tra cứu
    output = handler(**block.input)         # gọi hàm
    results.append(...)
```

Thêm công cụ = một mục trong mảng `TOOLS` + một dòng trong từ điển `TOOL_HANDLERS`. Vòng lặp vẫn giữ nguyên.

---

## Gọi Nhiều Công Cụ Cùng Lúc (Multiple Tool Calls)

Mô hình thường trả về nhiều yêu cầu `tool_use` trong cùng một lượt phản hồi — ví dụ: "đọc file a.py và b.py, sau đó liệt kê tất cả các file .py".

Các lệnh gọi công cụ được thực thi tuần tự từng cái một theo đúng thứ tự xuất hiện ban đầu trong `response.content`.

---

## Tham Khảo Nhanh

| Khái niệm | Tóm tắt một câu |
|-----------|-----------------|
| TOOL_HANDLERS | Từ điển ánh xạ Tên công cụ → hàm xử lý. Thêm công cụ = thêm một dòng ánh xạ |
| Định nghĩa Công cụ | JSON schema mô tả cho mô hình biết "tôi có thể làm được gì" |
| Gọi nhiều công cụ | Mô hình có thể trả về nhiều `tool_use` cùng lúc; các lệnh gọi thực thi theo thứ tự ban đầu |
| Vòng lặp không đổi | Vòng lặp `while True` từ s01 — không thay đổi dù chỉ một dòng |

---

## Thay Đổi So Với s01

| Thành phần | Trước (s01) | Sau (s02) |
|------------|-------------|-----------|
| Số lượng công cụ | 1 (bash) | 5 (+read, write, edit, glob) |
| Thực thi công cụ | Gọi cố định `run_bash()` | Tra cứu điều phối qua TOOL_HANDLERS |
| An toàn đường dẫn | Không có | Xác thực qua `safe_path` (chỉ cho các công cụ file) |
| Vòng lặp | `while True` + khối `tool_use` | Giống hệt s01 |

---

## Thử Nghiệm

```sh
cd learn-claude-code
python s02_tool_use/code.py
```

Hãy thử các câu lệnh mẫu sau:

1. `Read the file README.md and tell me what this project is about`
2. `Create a file called test.py that prints "hello", then read it back`
3. `Find all Python files in this directory`
4. `Read both README.md and requirements.txt, then create a summary file`

Điểm cần quan sát: Khi nào mô hình chỉ gọi một công cụ, và khi nào nó gọi nhiều công cụ cùng lúc? Các lệnh gọi công cụ liên tiếp có được thực thi theo đúng thứ tự không?

---

## Tiếp Theo

Agent hiện đã có 5 công cụ chuyên dụng. Các công cụ thao tác với file đã được bảo vệ bởi `safe_path`, nhưng công cụ bash thì vẫn chưa bị hạn chế — lệnh nguy hiểm như `rm -rf /` vẫn có thể chạy.

→ s03 Permission: Thêm một cổng kiểm duyệt trước khi thực thi công cụ — thao tác này có an toàn không? Có cần người dùng phê duyệt trước không?


<!-- translation-sync: zh@v1, en@v1, ja@v1 -->
