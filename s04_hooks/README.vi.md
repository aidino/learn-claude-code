# s04: Hooks — Gắn Ngoài Vòng Lặp, Đừng Viết Vào Bên Trong

[English](README.md) · [中文](README.zh.md) · [日本語](README.ja.md) · [Tiếng Việt](README.vi.md)

s01 → s02 → s03 → `s04` → [s05](../s05_todo_write/) → s06 → ... → s16 → s17

> *"Hang on the loop, don't write into it"* — Hooks tiêm thêm logic mở rộng vào trước và sau khi thực thi công cụ.
>
> **Lớp Harness**: Hooks — Các điểm mở rộng không xâm lấn vào vòng lặp cốt lõi.

---

## Vấn đề Đặt ra

Agent ở s03 đã có cơ chế kiểm tra phân quyền. Nhưng mỗi khi muốn thêm một bước kiểm tra mới, như "ghi log mọi lệnh bash", "tự động git add sau khi ghi file", bạn lại phải sửa trực tiếp hàm `agent_loop`.

Vòng lặp sẽ nhanh chóng biến thành thế này:

```python
def agent_loop(messages):
    while True:
        # ... gọi LLM ...
        for block in response.content:
            if block.type != "tool_use":
                continue
            log_to_file(block)          # thêm một dòng
            check_permission(block)     # thêm một dòng
            notify_slack(block)         # lại thêm một dòng
            output = execute(block)
            auto_git_add(block)         # lại thêm một dòng nữa
            # ... vòng lặp dần trở nên hỗn loạn
```

Thứ bạn muốn mở rộng là hành vi của Agent, nhưng thứ bạn đang sửa lại chính là vòng lặp. Vòng lặp phải luôn là một nhân cốt lõi ổn định; các logic mở rộng phải được móc nối từ bên ngoài.

---

## Giải pháp

![Hooks Overview](images/hooks-overview.en.svg)

Vòng lặp và logic phân quyền của s03 được bảo toàn nguyên vẹn. Thay đổi duy nhất là chuyển `check_permission()` từ bên trong thân vòng lặp sang gắn vào một hook. Vòng lặp không còn gọi trực tiếp bất kỳ hàm kiểm tra cụ thể nào nữa. Thay vào đó, nó gọi `trigger_hooks("PreToolUse", block)`, và bảng đăng ký hook (registry) sẽ quyết định những hàm nào được chạy.

Bốn sự kiện bao trọn một chu trình hoàn chỉnh của agent:

| Sự kiện | Thời điểm kích hoạt | Ứng dụng tiêu biểu |
|---------|---------------------|-------------------|
| UserPromptSubmit | Sau khi người dùng nhập liệu, trước khi gửi tới LLM | Xác thực dữ liệu đầu vào, tiêm ngữ cảnh (context injection) |
| PreToolUse | Trước khi thực thi công cụ | Kiểm tra phân quyền, ghi log |
| PostToolUse | Sau khi thực thi công cụ | Tác vụ phụ (tự động git add...), kiểm tra kết quả đầu ra |
| Stop | Khi vòng lặp chuẩn bị thoát | Dọn dẹp tài nguyên, quyết định xem vòng lặp có tiếp tục hay không |

Các logic mở rộng được thêm vào thông qua `register_hook()`. Vòng lặp chỉ việc gọi `trigger_hooks()`.

---

## Cơ chế Hoạt động

**Bảng đăng ký Hook**: một từ điển ánh xạ tên sự kiện tới danh sách các hàm gọi lại (callbacks).

```python
HOOKS = {
    "UserPromptSubmit": [],
    "PreToolUse": [],
    "PostToolUse": [],
    "Stop": [],
}

def register_hook(event: str, callback):
    HOOKS[event].append(callback)

def trigger_hooks(event: str, *args):
    for callback in HOOKS[event]:
        result = callback(*args)
        if result is not None:   # giá trị trả về ≠ None → hook yêu cầu "dừng"
            return result
    return None
```

Khi `PreToolUse` trả về giá trị khác `None`, lệnh thực thi công cụ hiện tại sẽ bị chặn. Khi `Stop` trả về giá trị khác `None`, vòng lặp sẽ tiếp tục chạy thay vì thoát. Các giá trị trả về từ `UserPromptSubmit` và `PostToolUse` không làm gián đoạn luồng điều khiển.

**UserPromptSubmit** kích hoạt sau khi người dùng nhập câu lệnh và trước khi gửi vào LLM. Hook dưới đây ghi lại thư mục làm việc hiện tại:

```python
def context_inject_hook(query: str) -> str | None:
    """Tiêm thông tin thư mục làm việc hiện tại vào mỗi prompt."""
    print(f"\033[90m[HOOK] UserPromptSubmit: working in {WORKDIR}\033[0m")
    return None   # trả về None = không thay đổi, cho prompt đi qua bình thường
```

Trong vòng lặp chính, kích hoạt ngay sau khi người dùng nhập liệu:

```python
query = input("s04 >> ")
trigger_hooks("UserPromptSubmit", query)   # ← trước khi gửi tới LLM
history.append({"role": "user", "content": query})
agent_loop(history)
```

**PreToolUse / PostToolUse**, các điểm can thiệp trước và sau khi thực thi công cụ. Logic kiểm tra phân quyền từ s03 nay được đóng gói thành một PreToolUse hook, kèm thêm một hook ghi log và một hook cảnh báo kết quả đầu ra quá lớn:

```python
# PreToolUse: kiểm tra phân quyền (logic s03, chuyển từ vòng lặp sang hook)
def permission_hook(block):
    if block.name == "bash":
        for pattern in DENY_LIST:
            if pattern in block.input.get("command", ""):
                return "Permission denied by deny list"
    if block.name in ("read_file", "write_file", "edit_file"):
        path = block.input.get("path", "")
        if not (WORKDIR / path).resolve().is_relative_to(WORKDIR):
            choice = input("   Allow? [y/N] ").strip().lower()
            if choice not in ("y", "yes"):
                return "Permission denied by user"
    return None

# PreToolUse: ghi log
def log_hook(block):
    print(f"[HOOK] {block.name}(...)")

# PostToolUse: nhắc nhở khi đầu ra quá lớn
def large_output_hook(block, output):
    if len(str(output)) > 100000:
        print(f"[HOOK] ⚠ Large output from {block.name}")

register_hook("PreToolUse", permission_hook)
register_hook("PreToolUse", log_hook)
register_hook("PostToolUse", large_output_hook)
```

**Stop** kích hoạt khi vòng lặp chuẩn bị kết thúc. Hook dưới đây in ra phần tóm tắt dọn dẹp phiên làm việc:

```python
def summary_hook(messages: list) -> str | None:
    """In phần tóm tắt khi vòng lặp chuẩn bị dừng."""
    tool_count = sum(1 for m in messages
                     for b in (m.get("content") if isinstance(m.get("content"), list) else [])
                     if isinstance(b, dict) and b.get("type") == "tool_result")
    print(f"\033[90m[HOOK] Stop: session used {tool_count} tool calls\033[0m")
    return None   # trả về None = cho phép dừng, trả về chuỗi = ép buộc tiếp tục

register_hook("Stop", summary_hook)
```

Trong agent_loop, kích hoạt trước khi thoát:

```python
tool_calls = [
    block for block in response.content if block.type == "tool_use"
]
if not tool_calls:
    force = trigger_hooks("Stop", messages)   # ← trước khi thoát
    if force:
        # hook trả về một thông điệp → chèn vào và tiếp tục
        messages.append({"role": "user", "content": force})
        continue
    return
```

**Chỉ có một thay đổi duy nhất trong vòng lặp**: s03 gọi trực tiếp `check_permission(block)`, s04 thay bằng `trigger_hooks("PreToolUse", block)`:

```python
for block in tool_calls:
    # s03: if not check_permission(block): ...
    # s04: hooks thay thế cho việc gọi cứng (hardcoding)
    blocked = trigger_hooks("PreToolUse", block)
    if blocked:
        results.append({"type": "tool_result", "tool_use_id": block.id,
                        "content": str(blocked)})
        continue

    handler = TOOL_HANDLERS.get(block.name)
    output = handler(**block.input) if handler else f"Unknown: {block.name}"

    trigger_hooks("PostToolUse", block, output)

    results.append({"type": "tool_result", "tool_use_id": block.id,
                        "content": output})
```

Bốn hook bao quát trọn vẹn các điểm nút quan trọng trong chu trình agent: đầu vào → trước thực thi → sau thực thi → thoát. Vòng lặp chỉ việc gọi trigger_hooks(); toàn bộ logic nghiệp vụ nằm gọn trong các hàm callback của hook.

---

## Thay Đổi So Với s03

| Thành phần | Trước (s03) | Sau (s04) |
|------------|-------------|-----------|
| Cơ chế mở rộng | check_permission() được gọi cứng trong vòng lặp | Bảng đăng ký HOOKS + trigger_hooks() |
| Các hàm mới | — | register_hook, trigger_hooks |
| Các hàm callback hook | — | context_inject_hook, permission_hook, log_hook, large_output_hook, summary_hook |
| Vòng lặp | Gọi trực tiếp check_permission() | Gọi trigger_hooks("PreToolUse", ...) |
| Kiểm soát thoát | Không có | trigger_hooks("Stop", ...) có thể ngăn thoát vòng lặp |
| Can thiệp đầu vào | Không có | trigger_hooks("UserPromptSubmit", ...) có thể tiêm ngữ cảnh |

---

## Thử Nghiệm

```sh
cd learn-claude-code
python s04_hooks/code.py
```

Hãy thử các câu lệnh mẫu sau:

1. `Read the file README.md` (chạy trực tiếp, quan sát log của hook)
2. `Create a file called test.txt` (sau khi tạo xong, quan sát xem PostToolUse có được kích hoạt không)
3. `Delete all temporary files in /tmp` (lệnh bash + rm kích hoạt permission hook)

Điểm cần quan sát: Trước mỗi lần thực thi công cụ, dòng log `[HOOK]` có xuất hiện không? Khi một thao tác bị từ chối phân quyền, nó bị chặn bởi hook hay do vòng lặp quy định cứng?

---

## Tiếp Theo

Giờ đây Agent đã có thể thực thi các thao tác một cách an toàn. Nhưng liệu nó có bao giờ dừng lại suy nghĩ xem "mình nên làm gì trước, làm gì sau" không? Trước một tác vụ phức tạp, liệu nó có nhảy bổ vào làm ngay hay sẽ lập kế hoạch trước?

→ s05 TodoWrite: Cung cấp cho Agent một công cụ lập kế hoạch. Tạo danh sách công việc trước, sau đó mới bắt tay thực thi.


<!-- translation-sync: zh@v1, en@v1, ja@v1 -->
