# s07: Skill Loading — Nạp Kỹ Năng Khi Có Nhu Cầu

[English](README.md) · [中文](README.zh.md) · [日本語](README.ja.md) · [Tiếng Việt](README.vi.md)

s01 → s02 → s03 → s04 → s05 → s06 → `s07` → [s08](../s08_context_compact/) → s09 → ... → s16 → s17

> System prompt chỉ chứa danh mục tổng quan kỹ năng (skill catalog); công cụ `load_skill` sẽ trả về toàn bộ nội dung file `SKILL.md`.
>
> **Lớp Harness**: Nạp tri thức (Knowledge loading) — cho mô hình biết những kỹ năng nào đang khả dụng, sau đó nạp chi tiết theo tên khi cần.

---

## Vấn đề Đặt ra

Giả sử một dự án có quy chuẩn xây dựng React component, hướng dẫn viết code SQL, và tài liệu thiết kế API. Chúng ta muốn Agent tuân thủ nghiêm ngặt các quy tắc này trong quá trình phát triển, nên cách tiếp cận trực tiếp nhất là nhét toàn bộ các tài liệu đó vào system prompt:

```python
SYSTEM = (
    f"You are a coding agent. "
    + open("docs/react-style.md").read()
    + open("docs/sql-style.md").read()
    + open("docs/api-design.md").read()
)
```

Cách làm này giúp Agent đọc được mọi quy chuẩn, nhưng nó lại cố định cả ba tài liệu trong system prompt thay vì chỉ chọn đúng tài liệu cần thiết cho tác vụ hiện tại. Mỗi lượt gọi LLM đều phải gửi nguyên văn cả ba tài liệu đến mô hình. Khi tác vụ chỉ đơn thuần là chỉnh sửa component React, chỉ có quy chuẩn React là thực sự liên quan; hướng dẫn viết SQL và thiết kế API vẫn tiếp tục tiêu tốn token đầu vào và chiếm dụng không gian cửa sổ ngữ cảnh (context window) — vốn dĩ nên được dành để chứa mã nguồn, lịch sử hội thoại và kết quả thực thi công cụ.

---

## Giải pháp

![Skill Overview](images/skill-overview.en.svg)

Khi khởi động, `SkillLoader` sẽ quét qua các thư mục `skills/*/SKILL.md`, đọc `name` và `description` từ phần YAML frontmatter, rồi đưa danh mục tổng quan đó vào system prompt. Khi mô hình cần hướng dẫn chi tiết đầy đủ, nó sẽ gọi `load_skill(name)`; nội dung `SKILL.md` trả về sẽ được nối vào danh sách tin nhắn dưới dạng một `tool_result`.

| Nội dung | Vị trí nạp vào mô hình | Thời điểm bổ sung |
|----------|------------------------|-------------------|
| Tên và mô tả kỹ năng | System prompt | Khi khởi động |
| Toàn văn `SKILL.md` | `tool_result` | Khi gọi hàm `load_skill` |

---

## Cơ chế Hoạt động

Mỗi kỹ năng được tổ chức thành một thư mục chứa file `SKILL.md`:

```text
skills/
  agent-builder/SKILL.md
  code-review/SKILL.md
  mcp-builder/SKILL.md
  pdf/SKILL.md
```

### Quét Kỹ Năng

```python
class SkillLoader:
    def scan(self):
        self.skills.clear()
        skills_root = self.skills_dir.resolve()
        for manifest in sorted(self.skills_dir.glob("*/SKILL.md")):
            if (not manifest.is_file()
                    or not manifest.resolve().is_relative_to(skills_root)):
                continue
            content = manifest.read_text(encoding="utf-8")
            metadata, body = self.parse_frontmatter(content)
            raw_name = metadata.get("name")
            name = raw_name.strip() if isinstance(raw_name, str) else ""
            name = name or manifest.parent.name
            raw_description = metadata.get("description")
            description = (raw_description.strip()
                           if isinstance(raw_description, str) else "")
            description = description or body.split("\n", 1)[0]
            description = " ".join(str(description).lstrip("# ").split())
            self.skills[name] = {
                "name": name,
                "description": description,
                "content": content,
            }
```

Hàm `catalog()` chỉ trả về tên và phần mô tả tóm tắt:

```text
- code-review: Perform thorough code reviews...
- pdf: Process PDF files...
```

### Xây dựng System Prompt

```python
ENVIRONMENT_PROMPT = (
    "Windows: the bash tool runs through cmd.exe; use cmd.exe syntax, not Unix "
    "Bash or PowerShell syntax, and prefer dedicated file tools for file operations"
    if os.name == "nt"
    else "Unix-like: the bash tool runs the system shell"
)

def build_system_prompt() -> str:
    return (
        f"You are a coding agent at {WORKDIR}. Environment: {ENVIRONMENT_PROMPT}. "
        "Use tools to solve tasks. "
        "Act, don't explain.\n\n"
        f"Skills available:\n{SKILL_LOADER.catalog()}\n\n"
        "Use load_skill to read the full instructions when a skill applies."
    )
```

Hàm này kết hợp các chỉ dẫn Agent cố định với danh mục kỹ năng tìm thấy khi khởi động.

### Nạp Toàn Văn Nội Dung

```python
def load(self, name: str) -> str:
    skill = self.skills.get(name)
    if skill:
        return skill["content"]
    available = ", ".join(self.skills) or "none"
    return f"Error: Unknown skill '{name}'. Available: {available}"
```

Tham số `name` được dùng để tra cứu trong bảng đăng ký lúc khởi động; nó không bị diễn giải thành một đường dẫn file. Sau khi công cụ trả về kết quả, Vòng lặp Agent hiện có sẽ nối thêm nội dung đó như một tin nhắn `tool_result` mới.

---

## Thử Nghiệm

```sh
cd learn-claude-code
python s07_skill_loading/code.py
```

Hãy thử các câu lệnh mẫu sau:

1. `What skills are available?`
2. `Load the code-review skill and follow its instructions`
3. `Review README.md and load the relevant skill first`

Kiểm tra để chắc chắn rằng system prompt chỉ chứa danh mục tóm tắt và toàn bộ nội dung `SKILL.md` chỉ xuất hiện sau khi lệnh `load_skill` được gọi.

---

## Tiếp Theo

Khi các lệnh gọi công cụ tích lũy ngày một nhiều, `messages[]` sẽ giữ lại toàn bộ nội dung file và kết quả công cụ trước đó.

→ s08 Context Compact: Thu gọn các tin nhắn cũ hơn và giữ cho ngữ cảnh luôn sẵn sàng cho các lượt gọi tiếp theo.


<!-- translation-sync: zh@v6, en@v6, ja@v6 -->
