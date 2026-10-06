# Đặc tả Thiết kế: Bản địa hóa toàn bộ Repository sang Tiếng Việt (Vietnamese Localization Design)

- **Ngày tạo**: 2026-10-06
- **Trạng thái**: Draft / Đề xuất duyệt
- **Mục tiêu**: Thiết lập hệ thống bản địa hóa tiếng Việt hoàn chỉnh cho `learn-claude-code`, đồng bộ về kiến trúc với tiếng Nhật (`ja`) và tiếng Trung (`zh`), bao gồm toàn bộ tài liệu Markdown, giao diện Web Next.js và quy trình trích xuất nội dung tự động.

---

## 1. Bối cảnh & Mục tiêu (Context & Objectives)

### 1.1 Hiện trạng repo
`learn-claude-code` là tài liệu học tập và khung tham chiếu kiến trúc mã nguồn mở về AI Coding Agent. Hiện repo đã hỗ trợ 3 ngôn ngữ:
- Tiếng Anh (`en` - ngôn ngữ gốc)
- Tiếng Trung (`zh`)
- Tiếng Nhật (`ja`)

Kiến trúc hiện tại được chuẩn hóa theo quy tắc:
1. **Mã nguồn (`code.py`)**: Giữ nguyên bằng tiếng Anh để đảm bảo cú pháp Python, tên thư viện và tính tương thích khi chạy code.
2. **Tài liệu Markdown**:
   - `README.md` ở root và các bản dịch `README-zh.md`, `README-ja.md`.
   - 17 thư mục bài học từ `s01_agent_loop` đến `s17_goal_loop`, mỗi thư mục có `README.md`, `README.zh.md`, `README.ja.md`.
   - Thư mục `docs/` chứa các bài viết chuyên sâu: `docs/en/`, `docs/zh/`, `docs/ja/` (mỗi ngôn ngữ 12 bài).
3. **Nền tảng Web (`web/`)**:
   - Ứng dụng Next.js đa ngôn ngữ định tuyến theo locale: `/[locale]/...`.
   - Từ điển dịch `web/src/i18n/messages/{en,zh,ja}.json`.
   - Script tự động `web/scripts/extract-content.ts` đọc các file markdown và code để build `docs.json` và `versions.json`.

### 1.2 Mục tiêu đề ra
- Bổ sung tiếng Việt (`vi`) thành ngôn ngữ chính thức thứ 4 trong repo.
- Đảm bảo 100% tài liệu lý thuyết, bài giảng, hướng dẫn và giao diện web có bản tiếng Việt chính xác, tự nhiên, văn phong kỹ thuật chuẩn mực.
- Cập nhật toàn bộ thanh điều hướng đa ngôn ngữ giữa các bài đọc.
- Giữ nguyên vẹn tính năng thực thi của các script code mẫu và quy trình build của web Next.js.

---

## 2. Phạm vi & Giới hạn (Scope & Non-Goals)

### 2.1 Trong phạm vi (In Scope)
1. **Tài liệu chính**:
   - `README-vi.md` tại thư mục gốc.
   - 17 file `README.vi.md` trong 17 thư mục bài học (`s01_agent_loop` đến `s17_goal_loop`).
   - 12 file bài viết chi tiết trong `docs/vi/s01-*.md` đến `docs/vi/s12-*.md`.
   - Cập nhật header language switcher trong tất cả các file `README*.md` hiện có:
     - Root: `[English](README.md) | [中文](README-zh.md) | [日本語](README-ja.md) | [Tiếng Việt](README-vi.md)`
     - Chapters: `[English](README.md) · [中文](README.zh.md) · [日本語](README.ja.md) · [Tiếng Việt](README.vi.md)`
2. **Nền tảng Web (`web/`)**:
   - Tạo mới `web/src/i18n/messages/vi.json` (dịch đầy đủ 117 keys/namespaces).
   - Đăng ký locale `"vi"` trong Next.js routing, metadata, context và header navigation.
   - Cập nhật `web/scripts/extract-content.ts` để nhận diện locale `vi` và cắt bỏ thanh điều hướng language header khi đưa nội dung vào web.
3. **Bảng thuật ngữ chuẩn (Glossary)**:
   - Xây dựng bảng quy chuẩn thuật ngữ Anh - Việt xuyên suốt để tránh dịch rời rạc hoặc tối nghĩa.

### 2.2 Ngoài phạm vi (Non-Goals)
- **Không dịch mã nguồn Python (`code.py`)**: Giữ nguyên tiếng Anh (chuẩn theo mô hình `zh` và `ja` hiện hành) để code chạy được ở mọi môi trường mà không lo lỗi encoding hoặc sai lệch cú pháp API Anthropic.
- **Không dịch lại các sơ đồ đồ họa nhị phân/SVG tĩnh**: Sử dụng sơ đồ kiến trúc mặc định (`.svg` và `.en.svg`), trừ khi có yêu cầu vẽ mới.
- **Không thay đổi cấu trúc thư mục hoặc logic của các module bài học**: Giữ nguyên tên thư mục `s01_agent_loop/`, cấu trúc `docs/`, `web/`.

---

## 3. Bảng Thuật Ngữ Kỹ Thuật Chuẩn (Glossary)

Để văn phong kỹ thuật nhất quán, tự nhiên và dễ hiểu đối với lập trình viên Việt Nam:

| Thuật ngữ gốc (English) | Bản dịch đề xuất | Ghi chú / Quy tắc |
|-------------------------|-------------------|-------------------|
| **Agent Loop** | Vòng lặp Agent / Chu trình Agent | Giữ thuật ngữ "Agent", dịch Loop là "vòng lặp" hoặc "chu trình" |
| **Tool Use / Tool Calling** | Sử dụng công cụ / Gọi công cụ | Giữ nguyên tên tool khi đề cập trong code (vd: `bash`, `read`) |
| **Tool Dispatch** | Điều phối công cụ | Bảng ánh xạ gọi hàm tương ứng với tool_use |
| **Permission Gate / Pipeline** | Cổng phân quyền / Đường ống phê duyệt | Phê duyệt lệnh trước khi thực thi |
| **Hooks / Lifecycle Hooks** | Điểm gắn vòng đời (Hooks) | Có thể dùng "Hooks" hoặc "Điểm can thiệp vòng đời" |
| **Subagent** | Subagent (Agent con / Agent phụ) | Giữ "Subagent" trong ngữ cảnh kỹ thuật |
| **Skill Loading** | Nạp kỹ năng / Tải kỹ năng | |
| **Context Compact / Compaction** | Thu gọn ngữ cảnh | Nén hoặc cắt gọt hội thoại để vừa context window |
| **Micro-compact** | Thu gọn vi mô (Micro-compact) | Cắt gọt từng phần nhỏ (vd: xóa output cũ) |
| **Auto-compact** | Tự động thu gọn | Kích hoạt tóm tắt khi vượt ngưỡng token |
| **Task System** | Hệ thống tác vụ | Quản lý DAG / danh sách công việc |
| **Background Tasks** | Tác vụ chạy ngầm | |
| **Cron Scheduler** | Bộ lập lịch định kỳ | Lập lịch theo biểu thức cron |
| **Agent Teams** | Đội ngũ Agent (Agent Teams) | Nhiều agent cộng tác theo vai trò |
| **Team Protocols** | Giao thức phối hợp nhóm | Quy ước giao tiếp giữa các agent |
| **Integrated Harness** | Khung điều hành tích hợp (Harness) | Phần mềm bọc ngoài quản lý agent |
| **Workflow Runtime** | Môi trường thực thi quy trình | |
| **Goal Loop** | Chu trình hướng mục tiêu | |
| **Prompt** | Lời nhắc (Prompt) | Có thể mở ngoặc giải thích ở lần đầu xuất hiện |
| **Token Budget** | Ngân sách token | Giới hạn số token khả dụng |
| **Turn** | Lượt (Lượt tương tác) | Một chu kỳ User <-> Assistant |

---

## 4. Chi Tiết Các File Cần Tạo & Chỉnh Sửa (File Inventory)

### 4.1 Tạo mới tài liệu Markdown (30 files)
1. `README-vi.md` (Root documentation)
2. 17 file Chapter READMEs:
   - `s01_agent_loop/README.vi.md`
   - `s02_tool_use/README.vi.md`
   - `s03_permission/README.vi.md`
   - `s04_hooks/README.vi.md`
   - `s05_todo_write/README.vi.md`
   - `s06_subagent/README.vi.md`
   - `s07_skill_loading/README.vi.md`
   - `s08_context_compact/README.vi.md`
   - `s09_memory/README.vi.md`
   - `s10_task_system/README.vi.md`
   - `s11_background_tasks/README.vi.md`
   - `s12_cron_scheduler/README.vi.md`
   - `s13_agent_teams/README.vi.md`
   - `s14_mcp_plugin/README.vi.md`
   - `s15_integrated_harness/README.vi.md`
   - `s16_workflow_runtime/README.vi.md`
   - `s17_goal_loop/README.vi.md`
3. 12 file Extended Guides trong `docs/vi/`:
   - `docs/vi/s01-the-agent-loop.md`
   - `docs/vi/s02-tool-use.md`
   - `docs/vi/s03-todo-write.md`
   - `docs/vi/s04-subagent.md`
   - `docs/vi/s05-skill-loading.md`
   - `docs/vi/s06-context-compact.md`
   - `docs/vi/s07-task-system.md`
   - `docs/vi/s08-background-tasks.md`
   - `docs/vi/s09-agent-teams.md`
   - `docs/vi/s10-team-protocols.md`
   - `docs/vi/s11-autonomous-agents.md`
   - `docs/vi/s12-worktree-task-isolation.md`

### 4.2 Cập nhật file Markdown hiện có (Header Language Switcher)
- Root files:
  - `README.md`
  - `README-zh.md`
  - `README-ja.md`
- 17 Chapter files (cả bản `.md`, `.zh.md`, `.ja.md`):
  Thêm liên kết `· [Tiếng Việt](README.vi.md)` vào dòng đầu tiên.

### 4.3 Tạo mới và chỉnh sửa phần Web UI (`web/`)
1. Tạo mới `web/src/i18n/messages/vi.json`:
   - Cung cấp bản dịch tiếng Việt cho 12 namespaces: `meta`, `nav`, `home`, `version`, `sim`, `timeline`, `layers`, `compare`, `diff`, `sessions`, `layer_labels`, `viz`.
2. Chỉnh sửa `web/src/lib/i18n.tsx`:
   - Import `vi from "@/i18n/messages/vi.json"`
   - Bổ sung `vi` vào `messagesMap = { en, zh, ja, vi }`.
3. Chỉnh sửa `web/src/lib/i18n-server.ts`:
   - Import `vi from "@/i18n/messages/vi.json"`
   - Bổ sung `vi` vào `messagesMap = { en, zh, ja, vi }`.
4. Chỉnh sửa `web/src/app/[locale]/layout.tsx`:
   - Cập nhật `const locales = ["en", "zh", "ja", "vi"];`
   - Bổ sung `vi` vào `metaMessages`.
5. Chỉnh sửa `web/src/components/layout/header.tsx`:
   - Cập nhật `LOCALES` array: thêm `{ code: "vi", label: "Tiếng Việt" }`.
6. Chỉnh sửa `web/scripts/extract-content.ts`:
   - Cập nhật `type Locale = "en" | "zh" | "ja" | "vi";`
   - Cập nhật mảng `locales: Locale[] = ["en", "zh", "ja", "vi"];` trong `buildRootDocs` và `buildLegacyDocs`.
   - Cập nhật regex strip language bar:
     ```typescript
     /^\[English\]\(README\.md\)\s*.\s*\[中文\]\(README\.zh\.md\)\s*.\s*\[日本語\]\(README\.ja\.md\)(\s*.\s*\[Tiếng Việt\]\(README\.vi\.md\))?\n\n?/m
     ```

---

## 5. Quy Trình Kiểm Thử & Đảm Bảo Chất Lượng (QA & Verification)

1. **Kiểm tra liên kết chéo (Markdown Link Verification)**:
   - Script kiểm tra không có liên kết hỏng giữa các bài viết (ví dụ trỏ sang `../s02_tool_use/` hoặc `README.vi.md`).
2. **Kiểm tra script trích xuất dữ liệu Web**:
   - Chạy `npm --prefix web run extract`.
   - Xác minh file `web/src/data/generated/docs.json` sinh ra đầy đủ các mục với `locale: "vi"` cho cả 17 chapters và 12 legacy docs.
3. **Kiểm tra TypeScript & Next.js Build**:
   - Chạy `npm --prefix web run build`.
   - Đảm bảo toàn bộ các route tĩnh `/[locale]/...` với locale `vi` được sinh ra thành công mà không có lỗi thiếu message key hay kiểu dữ liệu.
4. **Kiểm tra trải nghiệm trực quan (Visual & Navigation Check)**:
   - Khởi chạy dev server hoặc kiểm tra header xem nút chuyển ngữ "Tiếng Việt" hiển thị đúng và chuyển trang mượt mà.

---

## 6. Phân Kỳ Triển Khai (Phased Execution)

- **Giai đoạn 1: Nền tảng Web & Hệ Thống i18n**:
  - Tạo `web/src/i18n/messages/vi.json`.
  - Cập nhật cấu hình i18n (`i18n.tsx`, `i18n-server.ts`, `layout.tsx`, `header.tsx`, `extract-content.ts`).
  - Chạy thử `npm run extract` và `npm run build` để kiểm tra hạ tầng i18n.

- **Giai đoạn 2: Dịch Tài Liệu Cốt Lõi (Root & Chapters s01-s08)**:
  - Dịch `README-vi.md` ở root.
  - Dịch các chapter nền tảng: `s01_agent_loop` đến `s08_context_compact`.
  - Cập nhật liên kết chuyển ngữ trong các file này.

- **Giai đoạn 3: Dịch Tài Liệu Nâng Cao (Chapters s09-s17)**:
  - Dịch các chapter nâng cao: `s09_memory` đến `s17_goal_loop`.
  - Cập nhật liên kết chuyển ngữ tương ứng.

- **Giai đoạn 4: Dịch Thư Mục `docs/vi/` (12 Extended Guides)**:
  - Dịch toàn bộ 12 tài liệu từ `docs/en/` sang `docs/vi/`.

- **Giai đoạn 5: Tích hợp Toàn diện, Re-extract & Build Verification**:
  - Chạy `npm run extract` để cập nhật toàn bộ `docs.json`.
  - Chạy `npm run build` xác nhận trang web tĩnh được render không lỗi.
  - Review tổng thể văn phong, liên kết và hoàn tất bàn giao.
