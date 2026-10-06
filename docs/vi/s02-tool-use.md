# s02: Sử Dụng Công Cụ (Tool Use)

`s01 > [ s02 ] > s03 > s04 > s05 > s06 | s07 > s08 > s09 > s10 > s11 > s12`

> *"Adding a tool means adding one handler"* -- vòng lặp giữ nguyên không đổi; các công cụ mới chỉ cần đăng ký vào bảng điều phối (dispatch map).
>
> **Tầng Harness**: Điều phối công cụ (Tool dispatch) -- mở rộng phạm vi tiếp cận của mô hình.

## Vấn đề

Nếu chỉ có `bash`, agent phải chạy shell script cho mọi tác vụ. Lệnh `cat` cắt ngắn nội dung một cách khó lường, `sed` dễ lỗi với các ký tự đặc biệt, và mỗi lần gọi bash đều là một bề mặt rủi ro bảo mật không bị giới hạn. Các công cụ chuyên dụng như `read_file` và `write_file` cho phép bạn áp đặt cơ chế sandbox giới hạn đường dẫn trực tiếp ngay ở cấp độ công cụ.

Điểm mấu chốt: thêm công cụ không đòi hỏi phải thay đổi vòng lặp.

## Giải pháp

```
+--------+      +-------+      +------------------+
|  User  | ---> |  LLM  | ---> | Tool Dispatch    |
| prompt |      |       |      | {                |
+--------+      +---+---+      |   bash: run_bash |
                    ^           |   read: run_read |
                    |           |   write: run_wr  |
                    +-----------+   edit: run_edit |
                    tool_result | }                |
                                +------------------+

Bảng điều phối là một dict: {tool_name: handler_function}.
Một lần tra cứu duy nhất thay thế toàn bộ chuỗi if/elif rườm rà.
```

## Cách thức hoạt động

1. Mỗi công cụ có một hàm xử lý (handler function) riêng. Cơ chế sandbox kiểm tra đường dẫn ngăn ngừa việc thoát khỏi thư mục làm việc (workspace).

```python
def safe_path(p: str) -> Path:
    path = (WORKDIR / p).resolve()
    if not path.is_relative_to(WORKDIR):
        raise ValueError(f"Path escapes workspace: {p}")
    return path

def run_read(path: str, limit: int = None) -> str:
    text = safe_path(path).read_text(encoding="utf-8")
    lines = text.splitlines()
    if limit and limit < len(lines):
        lines = lines[:limit]
    return "\n".join(lines)[:50000]
```

2. Bảng điều phối liên kết tên công cụ với các hàm xử lý tương ứng.

```python
TOOL_HANDLERS = {
    "bash":       lambda **kw: run_bash(kw["command"]),
    "read_file":  lambda **kw: run_read(kw["path"], kw.get("limit")),
    "write_file": lambda **kw: run_write(kw["path"], kw["content"]),
    "edit_file":  lambda **kw: run_edit(kw["path"], kw["old_text"],
                                        kw["new_text"]),
}
```

3. Trong vòng lặp, tra cứu hàm xử lý theo tên. Phần thân của vòng lặp hoàn toàn không thay đổi so với s01.

```python
for block in response.content:
    if block.type == "tool_use":
        handler = TOOL_HANDLERS.get(block.name)
        output = handler(**block.input) if handler \
            else f"Unknown tool: {block.name}"
        results.append({
            "type": "tool_result",
            "tool_use_id": block.id,
            "content": output,
        })
```

Thêm một công cụ = thêm một hàm xử lý + thêm định nghĩa schema. Vòng lặp không bao giờ phải thay đổi.

## Những điểm thay đổi so với s01

| Thành phần       | Trước đây (s01)       | Sau khi cập nhật (s02)      |
|------------------|-----------------------|-----------------------------|
| Công cụ          | 1 (chỉ có bash)       | 4 (bash, read, write, edit) |
| Điều phối        | Gọi bash trực tiếp    | Dict `TOOL_HANDLERS`        |
| An toàn đường dẫn| Không có              | Sandbox qua `safe_path()`   |
| Vòng lặp Agent   | Không đổi             | Không đổi                   |

## Trải nghiệm thực tế

```sh
cd learn-claude-code
python agents/s02_tool_use.py
```

Hãy thử các prompt sau:

1. `Read the file requirements.txt`
2. `Create a file called greet.py with a greet(name) function`
3. `Edit greet.py to add a docstring to the function`
4. `Read greet.py to verify the edit worked`
