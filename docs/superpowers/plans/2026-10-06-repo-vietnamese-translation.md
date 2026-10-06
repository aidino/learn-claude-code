# Kế Hoạch Triển Khai: Bản Địa Hóa Toàn Bộ Repo Sang Tiếng Việt

> **Dành cho kỹ sư / agent thực thi:** SUB-SKILL BẮT BUỘC: Sử dụng `superpowers:subagent-driven-development` (khuyên dùng) hoặc `superpowers:executing-plans` để thực thi từng nhiệm vụ theo kế hoạch này. Các bước sử dụng cú pháp checkbox (`- [ ]`) để theo dõi tiến độ.

**Mục tiêu:** Bản địa hóa 100% tài liệu và nền tảng web của repo `learn-claude-code` sang tiếng Việt theo chuẩn kiến trúc của tiếng Nhật và tiếng Trung, bao gồm 1 root README, 17 chapter READMEs, 12 tài liệu mở rộng, giao diện i18n web Next.js và quy trình trích xuất tài liệu tự động.

**Kiến trúc:** Bổ sung locale `"vi"` song song với `"en"`, `"zh"`, `"ja"`. Tách bạch rõ rệt giữa tài liệu lý thuyết (Markdown + Web i18n) và mã nguồn thực thi (`code.py` giữ nguyên tiếng Anh). Đảm bảo pipeline `extract-content.ts` và `next build` hoạt động liền mạch không lỗi.

**Tech Stack:** Next.js (App Router, static rendering), TypeScript, React, Tailwind CSS, Markdown / GFM, Node.js / tsx.

**Spec:** `docs/superpowers/specs/2026-10-06-repo-vietnamese-translation-design.md`

## Ràng Buộc Chung (Global Constraints)

- Mã nguồn Python (`code.py`) trong các thư mục `s01_...` đến `s17_...` phải được giữ nguyên bằng tiếng Anh để đảm bảo tương thích khi chạy thực tế và khớp với AST parser của `web/scripts/extract-content.ts`.
- Tên file bản dịch phải tuân thủ chuẩn: `README-vi.md` ở root, `README.vi.md` ở từng chapter, và `docs/vi/sXX-*.md` cho tài liệu mở rộng.
- Mọi file README (gốc, chapters) phải có thanh chuyển ngữ đầy đủ 4 ngôn ngữ:
  - Root: `[English](README.md) | [中文](README-zh.md) | [日本語](README-ja.md) | [Tiếng Việt](README-vi.md)`
  - Chapters: `[English](README.md) · [中文](README.zh.md) · [日本語](README.ja.md) · [Tiếng Việt](README.vi.md)`
- Tuân thủ nghiêm ngặt Bảng thuật ngữ kỹ thuật (Glossary) trong tài liệu Spec.
- Toàn bộ 117 keys/namespaces trong `web/src/i18n/messages/vi.json` phải được dịch trọn vẹn, không để sót text tiếng Anh chưa dịch.

---

### Nhiệm Vụ 1: Hạ Tầng i18n & Giao Diện Web Tiếng Việt

**Files:**
- Tạo mới: `web/src/i18n/messages/vi.json`
- Chỉnh sửa: `web/src/lib/i18n.tsx:1-25`
- Chỉnh sửa: `web/src/lib/i18n-server.ts:1-15`
- Chỉnh sửa: `web/src/app/[locale]/layout.tsx:8-18`
- Chỉnh sửa: `web/src/components/layout/header.tsx:16-22`
- Chỉnh sửa: `web/scripts/extract-content.ts:20-25, 240-250, 345-350, 370-375`

**Interfaces:**
- Consumes: Cấu trúc JSON từ `web/src/i18n/messages/en.json` (12 namespaces: `meta`, `nav`, `home`, `version`, `sim`, `timeline`, `layers`, `compare`, `diff`, `sessions`, `layer_labels`, `viz`).
- Produces: Locale `"vi"` hợp lệ cho Next.js dynamic route `/[locale]`, context `I18nContext`, và script `extract-content.ts`.

- [ ] **Bước 1.1: Tạo file từ điển `web/src/i18n/messages/vi.json`**
  Dịch toàn bộ các chuỗi từ `en.json` sang tiếng Việt chuẩn kỹ thuật (các nhãn `nav`, `version`, `timeline`, `simulator`, `diff`, v.v.).

- [ ] **Bước 1.2: Đăng ký `vi` vào `web/src/lib/i18n.tsx` và `web/src/lib/i18n-server.ts`**
  Thêm `import vi from "@/i18n/messages/vi.json";` và cập nhật `const messagesMap: Record<string, Messages> = { en, zh, ja, vi };`.

- [ ] **Bước 1.3: Cập nhật route layout và language switcher**
  - Cập nhật `web/src/app/[locale]/layout.tsx`: `const locales = ["en", "zh", "ja", "vi"];` và `metaMessages`.
  - Cập nhật `web/src/components/layout/header.tsx`: thêm `{ code: "vi", label: "Tiếng Việt" }` vào `LOCALES`.

- [ ] **Bước 1.4: Cập nhật script `web/scripts/extract-content.ts`**
  - Đổi `type Locale = "en" | "zh" | "ja" | "vi";`.
  - Bổ sung `"vi"` vào `locales` trong `buildRootDocs` và `buildLegacyDocs`.
  - Cập nhật regex strip language header: `(\s*.\s*\[Tiếng Việt\]\(README\.vi\.md\))?`.

- [ ] **Bước 1.5: Kiểm tra xác thực (Validation)**
  Chạy lệnh: `npm --prefix web run extract`
  Kỳ vọng: Script chạy thành công, không có lỗi TypeScript, tạo ra `docs.json` và `versions.json`.

---

### Nhiệm Vụ 2: Dịch Tài Liệu Gốc (Root Documentation)

**Files:**
- Tạo mới: `README-vi.md`
- Chỉnh sửa: `README.md:1-5`
- Chỉnh sửa: `README-zh.md:1-5`
- Chỉnh sửa: `README-ja.md:1-5`

**Interfaces:**
- Consumes: Cấu trúc và nội dung của `README.md`.
- Produces: `README-vi.md` hoàn chỉnh đóng vai trò trang chủ hướng dẫn cho người học tiếng Việt.

- [ ] **Bước 2.1: Soạn thảo `README-vi.md`**
  - Dịch đầy đủ các phần: Giới thiệu, Triết lý thiết kế (17 bài học từ tối giản đến nâng cao), Lộ trình học (Learning Path), Bảng tóm tắt các chapter (s01 đến s17), Hướng dẫn cài đặt & chạy thử, và Tài nguyên tham khảo.
  - Sử dụng văn phong kỹ thuật mạch lạc, chuẩn xác theo Glossary.

- [ ] **Bước 2.2: Cập nhật liên kết ngôn ngữ ở đầu các file README root**
  Thêm `| [Tiếng Việt](README-vi.md)` vào thanh điều hướng trên cùng của `README.md`, `README-zh.md`, `README-ja.md` và `README-vi.md`.

- [ ] **Bước 2.3: Kiểm tra liên kết**
  Kiểm tra bằng lệnh grep hoặc regex để đảm bảo cú pháp liên kết không bị lỗi gõ hoặc hỏng URL.

---

### Nhiệm Vụ 3: Dịch Các Bài Học Nền Tảng (Chapters s01 - s08)

**Files:**
- Tạo mới:
  - `s01_agent_loop/README.vi.md` (Vòng lặp Agent tối giản)
  - `s02_tool_use/README.vi.md` (Cơ chế Tool Use & Dispatch)
  - `s03_permission/README.vi.md` (Phân quyền & Kiểm soát lệnh nguy hiểm)
  - `s04_hooks/README.vi.md` (Hệ thống Lifecycle Hooks)
  - `s05_todo_write/README.vi.md` (Quản lý tiến độ với TodoWrite)
  - `s06_subagent/README.vi.md` (Phân rã tác vụ với Subagent)
  - `s07_skill_loading/README.vi.md` (Cơ chế nạp Skill động theo yêu cầu)
  - `s08_context_compact/README.vi.md` (Thu gọn ngữ cảnh: Micro & Auto-compact)
- Chỉnh sửa: Dòng navigation header của `README.md`, `README.zh.md`, `README.ja.md` trong 8 thư mục trên.

**Interfaces:**
- Consumes: Các file `sXX_.../README.md` tương ứng.
- Produces: 8 file `README.vi.md` đạt chuẩn, đường dẫn ảnh trỏ đúng thư mục `images/`.

- [ ] **Bước 3.1: Dịch s01 đến s04 (Lớp Core Agent Loop & Security)**
  Tạo `README.vi.md` cho `s01_agent_loop`, `s02_tool_use`, `s03_permission`, `s04_hooks`.
- [ ] **Bước 3.2: Dịch s05 đến s08 (Lớp Planning, Subagent, Skill & Context)**
  Tạo `README.vi.md` cho `s05_todo_write`, `s06_subagent`, `s07_skill_loading`, `s08_context_compact`.
- [ ] **Bước 3.3: Cập nhật navigation header ở 8 chapter**
  Thêm ` · [Tiếng Việt](README.vi.md)` vào dòng đầu các file `README.md`, `README.zh.md`, `README.ja.md`, `README.vi.md`.
- [ ] **Bước 3.4: Chạy thử trích xuất Web**
  Chạy: `npm --prefix web run extract` để đảm bảo Markdown parse chuẩn và được nạp vào `docs.json`.

---

### Nhiệm Vụ 4: Dịch Các Bài Học Nâng Cao (Chapters s09 - s17)

**Files:**
- Tạo mới:
  - `s09_memory/README.vi.md` (Hệ thống bộ nhớ nhiều tầng)
  - `s10_task_system/README.vi.md` (Hệ thống Task DAG & Quản lý phụ thuộc)
  - `s11_background_tasks/README.vi.md` (Tác vụ chạy ngầm bất đồng bộ)
  - `s12_cron_scheduler/README.vi.md` (Bộ lập lịch tác vụ định kỳ)
  - `s13_agent_teams/README.vi.md` (Cộng tác đa Agent theo vai trò)
  - `s14_mcp_plugin/README.vi.md` (Giao thức Model Context Protocol - MCP)
  - `s15_integrated_harness/README.vi.md` (Khung điều hành tích hợp hoàn chỉnh)
  - `s16_workflow_runtime/README.vi.md` (Môi trường thực thi quy trình làm việc)
  - `s17_goal_loop/README.vi.md` (Chu trình tự đánh giá và hướng mục tiêu)
- Chỉnh sửa: Dòng navigation header của các file README trong 9 thư mục trên.

**Interfaces:**
- Consumes: Các file `sXX_.../README.md` tương ứng.
- Produces: 9 file `README.vi.md` hoàn chỉnh.

- [ ] **Bước 4.1: Dịch s09 đến s12 (Lớp Memory & Scheduling)**
  Tạo `README.vi.md` cho `s09_memory`, `s10_task_system`, `s11_background_tasks`, `s12_cron_scheduler`.
- [ ] **Bước 4.2: Dịch s13 đến s17 (Lớp Multi-Agent, MCP & Production Harness)**
  Tạo `README.vi.md` cho `s13_agent_teams`, `s14_mcp_plugin`, `s15_integrated_harness`, `s16_workflow_runtime`, `s17_goal_loop`.
- [ ] **Bước 4.3: Cập nhật navigation header ở 9 chapter**
  Thêm ` · [Tiếng Việt](README.vi.md)` vào tất cả các README liên quan.
- [ ] **Bước 4.4: Kiểm tra trích xuất dữ liệu Web**
  Chạy: `npm --prefix web run extract` để xác nhận tất cả 17 chapters đã có mục `locale: "vi"`.

---

### Nhiệm Vụ 5: Dịch 12 Bài Viết Chuyên Sâu (`docs/vi/`)

**Files:**
- Tạo mới thư mục: `docs/vi/`
- Tạo mới 12 files:
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

**Interfaces:**
- Consumes: 12 file trong `docs/en/*.md`.
- Produces: 12 bài viết tiếng Việt chuyên sâu trong `docs/vi/*.md`, tích hợp tự động vào trang đọc tài liệu của web Next.js.

- [ ] **Bước 5.1: Dịch đợt 1 (s01 đến s06)**
  Biên dịch 6 bài viết lý thuyết đầu tiên sang tiếng Việt kỹ thuật chuẩn xác.
- [ ] **Bước 5.2: Dịch đợt 2 (s07 đến s12)**
  Biên dịch 6 bài viết về hệ thống tác vụ, đội ngũ agent, tính tự chủ và cô lập git worktree.
- [ ] **Bước 5.3: Chạy trích xuất nội dung legacy docs**
  Chạy: `npm --prefix web run extract`
  Kiểm tra: `docs.json` ghi nhận đầy đủ 12 bài viết tiếng Việt trong `docs/vi/`.

---

### Nhiệm Vụ 6: Kiểm Thử Toàn Diện, Build Web & Nghiệm Thu

**Files:**
- Kiểm tra toàn bộ repo

**Interfaces:**
- Consumes: Toàn bộ các file Markdown và hệ thống i18n vừa tạo.
- Produces: Báo cáo xác thực build thành công 100%, không gãy link, không thiếu key i18n.

- [ ] **Bước 6.1: Chạy kiểm tra liên kết Markdown**
  Chạy throwaway script kiểm tra toàn bộ file `.md` và `.vi.md` để đảm bảo không có đường dẫn tương đối bị hỏng.

- [ ] **Bước 6.2: Chạy kiểm tra trích xuất tài nguyên**
  Lệnh: `npm --prefix web run extract`
  Xác minh: File `web/src/data/generated/docs.json` có đủ 17 chapters + 12 docs với `locale: "vi"`.

- [ ] **Bước 6.3: Chạy Build ứng dụng Web Next.js**
  Lệnh: `npm --prefix web run build`
  Xác minh: Toàn bộ các trang `/vi`, `/vi/s01` đến `/vi/s17`, `/vi/timeline`, `/vi/compare`, `/vi/layers` đều được biên dịch tĩnh (SSG) thành công không cảnh báo hoặc lỗi.

- [ ] **Bước 6.4: Smoke test giao diện Web**
  Kiểm tra switcher ngôn ngữ trên header, thanh bên sidebar, các nút điều hướng và nội dung hiển thị tiếng Việt mượt mà.
