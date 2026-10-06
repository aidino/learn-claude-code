# s03: Permission — Kiểm Tra Phân Quyền Trước Khi Thực Thi

[English](README.md) · [中文](README.zh.md) · [日本語](README.ja.md) · [Tiếng Việt](README.vi.md)

s01 → s02 → `s03` → [s04](../s04_hooks/) → s05 → ... → s16 → s17
> *"Check permissions before executing"* — Đường ống phân quyền (permission pipeline) quyết định thao tác nào cần được phê duyệt.
>
> **Lớp Harness**: Phân quyền (Permission) — một cổng kiểm soát trước khi thực thi công cụ.

---

## Vấn đề Đặt ra

Agent ở s02 có 5 công cụ. Các công cụ thao tác với file đã được bảo vệ bởi `safe_path`, nhưng bash thì hoàn toàn không bị hạn chế. Nếu bạn yêu cầu nó "hãy dọn dẹp dự án", nó có thể chạy lệnh hủy diệt `rm -rf /`.

An toàn bảo mật không thể dựa vào việc tin tưởng mô hình — nó cần được đảm bảo bằng code: một bước kiểm tra trước mỗi lần thực thi công cụ.

---

## Giải pháp

![Permission Overview](images/permission-overview.en.svg)

Vòng lặp từ s02 được giữ nguyên hoàn toàn. Thay đổi duy nhất là chèn thêm `check_permission()` trước khi thực thi công cụ — mỗi lệnh gọi công cụ sẽ đi qua ba cổng kiểm soát theo thứ tự cố định: từ chối tuyệt đối trước (hard deny), sau đó hỏi ý kiến mềm (soft ask), và nếu không khớp cổng nào thì cho phép thực thi.

Ba cổng tương ứng với ba cấp quyết định:

| Cổng | Mục đích | Khi Khớp Điều Kiện |
|------|----------|-------------------|
| 1. Danh sách cấm (Deny List) | Các thao tác bị cấm vĩnh viễn (`rm -rf /`, `sudo`) | Bị từ chối ngay lập tức, không thực thi |
| 2. Khớp quy tắc (Rule Matching) | Các thao tác phụ thuộc ngữ cảnh (đọc/ghi ngoài workspace, xóa file bằng `rm`) | Chuyển tiếp sang Cổng 3 |
| 3. Người dùng phê duyệt (User Approval) | Sau khi Cổng 2 khớp, tạm dừng để chờ người dùng xác nhận | Người dùng quyết định cho phép hay từ chối |

Nếu không khớp bất kỳ cổng nào trong ba cổng trên → thực thi trực tiếp. Hầu hết các thao tác an toàn thông thường đều đi theo lộ trình này.

---

## Cơ chế Hoạt động

![Permission Pipeline](images/permission-pipeline.en.svg)

**Cổng 1**: Danh sách cấm tuyệt đối (hard deny list). Kiểm tra trước tiên; nếu khớp, trả về thông báo chặn. Danh sách này dùng so khớp chuỗi đơn giản để minh họa vị trí đặt cổng phân quyền; đây không phải là một ranh giới bảo mật toàn diện.

```python
DENY_LIST = [
    "rm -rf /", "sudo", "shutdown", "reboot",
    "mkfs", "dd if=", "> /dev/sda",
]

def check_deny_list(command: str) -> str | None:
    for pattern in DENY_LIST:
        if pattern in command:
            return f"Blocked: '{pattern}' is on the deny list"
    return None
```

**Cổng 2**: Khớp quy tắc — định nghĩa "khi nào cần hỏi người dùng". Mỗi quy tắc chỉ định công cụ áp dụng và điều kiện kiểm tra.

```python
import re

DESTRUCTIVE_COMMAND_WORD = re.compile(
    r"(?i)(?:^|[;&|()\n])\s*(?:rm|del)(?=\s|$|[;&|()])"
)

def contains_destructive_command(command: str) -> bool:
    return bool(DESTRUCTIVE_COMMAND_WORD.search(command))

PERMISSION_RULES = [
    {
        "tools": ["read_file", "write_file", "edit_file"],
        "check": lambda args: not (WORKDIR / args.get("path", "")).resolve().is_relative_to(WORKDIR),
        "message": "Access outside workspace",
    },
    {
        "tools": ["bash"],
        "check": lambda args: contains_destructive_command(args.get("command", "")) or any(
            kw in args.get("command", "") for kw in ["rm ", "> /etc/", "chmod 777"]
        ),
        "message": "Potentially destructive command",
    },
]

def check_rules(tool_name: str, args: dict) -> str | None:
    for rule in PERMISSION_RULES:
        if tool_name in rule["tools"] and rule["check"](args):
            return rule["message"]
    return None
```

**Cổng 3**: Khi quy tắc khớp điều kiện, tạm dừng chờ người dùng nhập lệnh xác nhận.

```python
def ask_user(tool_name: str, args: dict, reason: str) -> str:
    print(f"\n⚠  {reason}")
    print(f"   Tool: {tool_name}({args})")
    choice = input("   Allow? [y/N] ").strip().lower()
    return "allow" if choice in ("y", "yes") else "deny"
```

**Ghép cả ba cổng lại với nhau**, chèn vào trước bước thực thi công cụ:

```python
def check_permission(block) -> bool:
    # Cổng 1: Danh sách cấm tuyệt đối
    if block.name == "bash":
        reason = check_deny_list(block.input.get("command", ""))
        if reason:
            print(f"\n⛔ {reason}")
            return False

    # Cổng 2 + 3: Khớp quy tắc → Người dùng phê duyệt
    reason = check_rules(block.name, block.input)
    if reason:
        decision = ask_user(block.name, block.input, reason)
        if decision == "deny":
            return False

    return True

# Trong agent_loop — vòng lặp của s02 chỉ thêm đúng một dòng kiểm tra:
for block in tool_calls:
    if not check_permission(block):           # ← MỚI
        results.append({... "content": "Permission denied."})
        continue
    output = TOOL_HANDLERS[block.name](**block.input)  # gốc s02
    results.append(...)
```

---

## Thay Đổi So Với s02

| Thành phần | Trước (s02) | Sau (s03) |
|------------|-------------|-----------|
| Mô hình bảo mật | Không có (tin tưởng mô hình tuyệt đối) | Đường ống phân quyền ba cổng |
| Các hàm mới | — | check_deny_list, check_rules, ask_user, check_permission |
| Vòng lặp | Thực thi tất cả công cụ trực tiếp | Chèn check_permission() trước khi thực thi |

---

## Thử Nghiệm

```sh
cd learn-claude-code
python s03_permission/code.py
```

Hãy thử các câu lệnh mẫu sau:

1. `Create a file called test.txt in the current directory` (thao tác an toàn, vượt qua kiểm tra)
2. `Delete the file test.txt` (bash + rm kích hoạt Cổng 2)
3. `What files are in the current directory?` (chỉ đọc, tất cả đều vượt qua)
4. `Try to write a file to /etc/something` (ghi ra ngoài workspace kích hoạt Cổng 2)
5. Trên Windows, `del test.txt` và `DEL test.txt` kích hoạt Cổng 2, trong khi các từ như `model`, `delimiter`, và `echo del test.txt` thì không.

Điểm cần quan sát: Những thao tác nào được đi thẳng? Thao tác nào cần bạn xác nhận? Thao tác nào bị từ chối tuyệt đối?

---

## Tiếp Theo

Hệ thống phân quyền đã đi vào hoạt động — nhưng mọi thao tác kiểm tra đều đang được gọi cứng (hardcode) dưới dạng `check_permission()` bên trong vòng lặp. Nếu bạn muốn thêm việc ghi log trước và sau khi gọi công cụ thì sao? Nếu muốn tự động tạo git commit sau một số thao tác thì sao? Rải rác các logic mở rộng này khắp vòng lặp sẽ khiến nó nhanh chóng phình to.

→ s04 Hooks: Bổ sung các điểm can thiệp vòng đời (hooks) vào vòng lặp. Toàn bộ logic mở rộng sẽ được gắn vào hooks; vòng lặp vẫn luôn gọn gàng và tinh giản.


<!-- translation-sync: zh@v1, en@v1, ja@v1 -->
