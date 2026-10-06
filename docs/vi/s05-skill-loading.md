# s05: Nạp Kỹ Năng (Skill Loading)

`s01 > s02 > s03 > s04 > [ s05 ] > s06 | s07 > s08 > s09 > s10 > s11 > s12`

> *"Load knowledge when you need it, not upfront"* -- nạp tri thức khi cần thông qua tool_result, thay vì nhồi nhét tất cả vào system prompt ngay từ đầu.
>
> **Tầng Harness**: Tri thức theo yêu cầu (On-demand knowledge) -- chuyên môn theo từng lĩnh vực, chỉ tải khi mô hình yêu cầu.

## Vấn đề

Bạn muốn agent tuân thủ các quy trình làm việc đặc thù theo từng lĩnh vực: quy ước commit git, mẫu kiểm thử chuẩn, danh sách kiểm tra khi review code (code review checklists). Việc đưa tất cả những tri thức này vào system prompt sẽ lãng phí token cho những kỹ năng không dùng tới. Giả sử 10 kỹ năng tiêu tốn 2.000 token mỗi kỹ năng = 20.000 token, mà phần lớn trong số đó hoàn toàn không liên quan đến tác vụ hiện tại.

## Giải pháp

```
System prompt (Tầng 1 -- luôn luôn hiện diện):
+--------------------------------------+
| You are a coding agent.              |
| Skills available:                    |
|   - git: Git workflow helpers        |  ~100 tokens/skill
|   - test: Testing best practices     |
+--------------------------------------+

Khi mô hình gọi load_skill("git"):
+--------------------------------------+
| tool_result (Tầng 2 -- theo yêu cầu):|
| <skill name="git">                   |
|   Full git workflow instructions...  |  ~2000 tokens
|   Step 1: ...                        |
| </skill>                             |
+--------------------------------------+
```

Tầng 1: *tên* và mô tả ngắn của kỹ năng nằm trong system prompt (chi phí token rất rẻ). Tầng 2: toàn bộ *nội dung* chi tiết nạp qua tool_result (chỉ tải khi thực sự cần).

## Cách thức hoạt động

1. Mỗi kỹ năng là một thư mục chứa file `SKILL.md` có phần tiêu đề YAML (frontmatter).

```
skills/
  pdf/
    SKILL.md       # ---\n name: pdf\n description: Process PDF files\n ---\n ...
  code-review/
    SKILL.md       # ---\n name: code-review\n description: Review code\n ---\n ...
```

2. `SkillLoader` quét tìm các file `SKILL.md`, sử dụng tên thư mục làm định danh kỹ năng.

```python
class SkillLoader:
    def __init__(self, skills_dir: Path):
        self.skills = {}
        for f in sorted(skills_dir.rglob("SKILL.md")):
            text = f.read_text(encoding="utf-8")
            meta, body = self._parse_frontmatter(text)
            name = meta.get("name", f.parent.name)
            self.skills[name] = {"meta": meta, "body": body}

    def get_descriptions(self) -> str:
        lines = []
        for name, skill in self.skills.items():
            desc = skill["meta"].get("description", "")
            lines.append(f"  - {name}: {desc}")
        return "\n".join(lines)

    def get_content(self, name: str) -> str:
        skill = self.skills.get(name)
        if not skill:
            return f"Error: Unknown skill '{name}'."
        return f"<skill name=\"{name}\">\n{skill['body']}\n</skill>"
```

3. Tầng 1 được đưa vào system prompt. Tầng 2 chỉ đơn thuần là một hàm xử lý công cụ khác trong bảng điều phối.

```python
SYSTEM = f"""You are a coding agent at {WORKDIR}.
Skills available:
{SKILL_LOADER.get_descriptions()}"""

TOOL_HANDLERS = {
    # ...base tools...
    "load_skill": lambda **kw: SKILL_LOADER.get_content(kw["name"]),
}
```

Mô hình biết được những kỹ năng nào đang tồn tại (chi phí rẻ) và chỉ tải nội dung chi tiết khi thấy phù hợp với tác vụ (tiết kiệm chi phí).

## Những điểm thay đổi so với s04

| Thành phần       | Trước đây (s04)   | Sau khi cập nhật (s05)          |
|------------------|-------------------|---------------------------------|
| Công cụ          | 5 (cơ sở + task)  | 5 (cơ sở + load_skill)          |
| System prompt    | Chuỗi cố định     | + danh sách mô tả kỹ năng       |
| Tri thức miền    | Không có          | Các file skills/\*/SKILL.md     |
| Cơ chế nạp       | Không có          | Hai tầng (system prompt + result)|

## Trải nghiệm thực tế

```sh
cd learn-claude-code
python agents/s05_skill_loading.py
```

Hãy thử các prompt sau:

1. `What skills are available?`
2. `Load the agent-builder skill and follow its instructions`
3. `I need to do a code review -- load the relevant skill first`
4. `Build an MCP server using the mcp-builder skill`
