# s15: Khung Điều Hành Tích Hợp — Nhiều Cơ Chế, Một Vòng Lặp

[English](README.md) · [中文](README.zh.md) · [日本語](README.ja.md) · [Tiếng Việt](README.vi.md)

s01 → ... → s13 → [s14](../s14_mcp_plugin/) → `s15` → [s16](../s16_workflow_runtime/) → s17

> *"Nhiều cơ chế, một vòng lặp"* — các công cụ, phân quyền, bộ nhớ, tác vụ, đội ngũ và plugin đều gắn kết vào cùng một vòng lặp `while True`.
>
> **Lớp Harness**: Tích hợp (Integration) — tập hợp các cơ chế được sử dụng trong các ví dụ trước vào một hệ thống duy nhất có thể chạy được.

---

## Vấn Đề Đặt Ra

Các chương trước giữ các cơ chế riêng biệt trong từng ví dụ độc lập. Chương này kết nối tất cả các cơ chế cần thiết cho một môi trường thực thi tích hợp.

Một AI coding agent chạy đường dài cần tất cả các yếu tố này cùng một lúc:

- Điều phối công cụ và ranh giới phân quyền
- Các điểm mở rộng hook
- Lập kế hoạch todo và đồ thị tác vụ
- Kỹ năng, bộ nhớ và việc tự động tập hợp system prompt khi chạy
- Thu gọn ngữ cảnh và khôi phục lỗi
- Tác vụ chạy ngầm và lập lịch cron
- Đội ngũ agent, giao thức phối hợp và việc tự động nhận tác vụ khi IDLE
- Worktree liên kết theo từng tác vụ
- Tích hợp công cụ bên ngoài qua MCP

S15 không giới thiệu thêm một cơ chế cô lập mới nào. Nó chỉ ra vị trí các cơ chế hiện có tham gia vào vòng lặp mô hình và cách các sự kiện của chúng quay trở lại cùng một cuộc hội thoại.

---

## Giải Pháp

![System Architecture](images/system-architecture.en.svg)

S15 không tạo ra cơ chế mới mà kết nối các thành phần từ các chương trước vào một khung điều hành tích hợp:

```text
đầu vào người dùng
  → hooks UserPromptSubmit
  → chèn thông báo cron/chạy ngầm
  → thu gọn ngữ cảnh (context compact)
  → bộ nhớ + kỹ năng + trạng thái MCP tập hợp system prompt
  → LLM
  → có khối tool_use không?
      không → hooks Stop → trả về kết quả
      có    → hooks PreToolUse + phân quyền
            → TOOL_HANDLERS / MCP handlers / điều phối chạy ngầm
            → hooks PostToolUse
            → đưa tool_result / task_notification ngược lại messages
            → vòng lặp tiếp theo
```

Vòng lặp vẫn giữ nguyên cấu trúc cốt lõi: gọi mô hình, kiểm tra xem phản hồi có chứa khối `tool_use` hay không, thực thi các công cụ, và nối kết quả vào `messages`. Sự hiện diện của khối `tool_use` quyết định việc thực thi công cụ có tiếp tục hay không.

---

## Vị Trí Của Từng Thành Phần

| Vị trí | Thành phần | Vai trò |
|---|---|---|
| Xung quanh đầu vào người dùng | hooks `UserPromptSubmit` | Ghi log, chèn thêm thông tin, hoặc kiểm toán đầu vào |
| Trước khi gọi LLM | hàng đợi cron | Chèn các prompt theo lịch vào `messages` |
| Trước khi gọi LLM | thông báo chạy ngầm | Chèn kết quả chạy ngầm đã hoàn thành dưới dạng `<task_notification>` |
| Trước khi gọi LLM | đường ống thu gọn ngữ cảnh | Kiểm soát ngân sách output lớn, cắt gọt lịch sử, nén kết quả công cụ cũ, tóm tắt khi cần |
| Trước khi gọi LLM | bộ nhớ / kỹ năng / trạng thái MCP | Tập hợp system prompt để mô hình thấy năng lực hiện tại và ngữ cảnh dài hạn |
| Lệnh gọi LLM | khôi phục lỗi | Thử lại lỗi 429/529, tăng dần `max_tokens`, thu gọn ngữ cảnh khi prompt quá dài |
| Trước khi chạy công cụ | hooks `PreToolUse` + phân quyền | Chặn các lệnh nguy hiểm, ghi file ngoài phạm vi, công cụ MCP có tính phá hủy |
| Điều phối công cụ | `assemble_tool_pool` | Tập hợp các công cụ tích hợp sẵn và công cụ MCP động |
| Trong khi chạy công cụ | điều phối chạy ngầm | Chuyển lệnh bash được đánh dấu sang luồng daemon và trả về kết quả giữ chỗ |
| Sau khi chạy công cụ | hooks `PostToolUse` | Cảnh báo output lớn, ghi log, hậu xử lý |
| Quay lại vòng lặp | tool_result | Mỗi `tool_use` nhận một `tool_result`, sau đó chuyển sang vòng gọi mô hình tiếp theo |
| Không có tool_use / khi dừng | hooks `Stop` | Thống kê, dọn dẹp tài nguyên, kiểm toán |

---

## Những Gì File code.py Chứa Đựng

### Công Cụ Và Điều Phối

Kho công cụ tích hợp sẵn bao gồm 26 công cụ:

```text
bash, read_file, write_file, edit_file, glob
todo_write, task, load_skill, compact
create_task, update_task, list_tasks, get_task, claim_task, complete_task
schedule_cron, list_crons, cancel_cron
spawn_teammate, list_teammates, send_message
request_shutdown, request_plan, review_plan
create_worktree
connect_mcp
```

Hàm `assemble_tool_pool()` tập hợp các công cụ này ở mỗi vòng:

```text
BUILTIN_TOOLS + các công cụ MCP đang kết nối
BUILTIN_HANDLERS + các hàm xử lý mcp__server__tool
```

Sau khi gọi `connect_mcp("docs")`, vòng tiếp theo sẽ hiển thị thêm các công cụ như `mcp__docs__search`.

### Phân Quyền Và Hooks

Phân quyền không bị gắn cứng vào dòng thực thi công cụ. Nó là một hook `PreToolUse`:

```python
blocked = trigger_hooks("PreToolUse", block)
if blocked:
    results.append(tool_result(block.id, blocked))
    continue
```

Điều đó có nghĩa là phân quyền, ghi log, và kiểm toán đều gắn vào cùng một điểm hook. Các công cụ của Lead, công cụ subagent dùng một lần, và công cụ của teammate đều đi qua `PreToolUse`; một lệnh gọi được phép sẽ chạy hook `PostToolUse` sau khi hàm xử lý của nó hoàn tất.

Chính sách không tin tưởng mô tả của chính server MCP như một sự cấp quyền. Host nắm giữ một danh sách cho phép chính xác nhỏ cho các lệnh gọi chỉ đọc đã biết; mọi công cụ MCP khác đều hỏi ý kiến người dùng. Các công cụ file bị từ chối ngoài `WORKDIR`, và mọi lệnh bash đều hỏi trước khi thực thi. Chỉ lượt tương tác trực tiếp của người dùng ở tiền cảnh mới có thể mở lời nhắc phê duyệt tương tác; các lượt chạy bất đồng bộ sẽ từ chối tự động thay vì cạnh tranh đầu vào stdin với terminal chính.

### Lập Kế Hoạch Và Tác Vụ

S15 duy trì hai tầng lập kế hoạch:

- `todo_write`: kế hoạch nhẹ cho phiên hiện tại, lưu trong bộ nhớ
- Đồ thị tác vụ: các file tác vụ xuyên phiên, có nhận biết phụ thuộc, có thể nhận việc lưu dưới `.tasks/task_*.json`

Cơ chế đầu tiên giúp một agent đơn lẻ không bị chệch hướng. Cơ chế thứ hai hỗ trợ sự phối hợp trong đội ngũ.

Chúng chia sẻ mục đích chứ không chia sẻ cách triển khai: `todo_write` thay thế danh sách kiểm tra của một phiên, trong khi các bản ghi tác vụ có ID ổn định và các cập nhật vòng đời riêng lẻ. Công cụ `task` riêng biệt bên dưới có nghĩa là "điều phối một subagent độc lập"; nó không phải là Hệ Thống Tác Vụ (Task System).

Việc xây dựng đồ thị tác vụ vẫn giữ nguyên quy trình hai giai đoạn trong host tích hợp: Lead tạo tất cả các nút tác vụ trước, sau đó gọi `update_task` với các ID thời gian chạy do `create_task` trả về. Các teammate chỉ nhận các thao tác list, claim và complete, do đó cấu trúc phụ thuộc được Lead cố định trước khi công việc được phân phát.

### Subagent Và Đội Ngũ

S15 có hai hình thức ủy quyền:

- `task`: subagent dùng một lần. Nó sử dụng một mảng `messages[]` riêng biệt, hủy bỏ ngữ cảnh trung gian, và chỉ trả về bản tóm tắt cuối cùng.
- `spawn_teammate`: luồng teammate bền vững. Khi được truyền một `task_id` sẵn sàng, runtime sẽ nhận task đó trước khi luồng bắt đầu; nếu không có, teammate có thể chờ ở trạng thái IDLE để nhận việc sau. Một teammate chưa được phân công nhiệm vụ không thể dùng các công cụ file hay Shell. Nó tuân theo chu trình `WORK → result → IDLE` mà không bị giới hạn số vòng công cụ cố định; lỗi mô hình hoặc điều phối sẽ phát ra một `error`, và việc dọn dẹp luồng sẽ giải phóng nhiệm vụ chưa xong trở lại bảng tác vụ. Nó làm rỗng hộp thư trước mỗi lệnh gọi mô hình, vì vậy tin nhắn trực tiếp và yêu cầu dừng hệ thống không bị xếp hàng sau một chuỗi gọi công cụ liên tục. Khi rảnh rỗi, trước tiên nó chờ nhận tin từ `MessageBus`, chỉ sau khi hết thời gian chờ mới quét các tác vụ sẵn sàng và nhận nguyên tử tối đa một tác vụ.

Sau khi khởi tạo một teammate, Lead kết thúc lượt hiện tại thay vì liên tục truy vấn trạng thái của nó bên trong vòng lặp mô hình. Một sự kiện nhóm trong hộp thư của Lead sẽ kích hoạt runtime bắt đầu lượt tiếp theo.

Subagent dùng một lần giải quyết vấn đề cách ly ngữ cảnh. Teammate bền vững giải quyết vấn đề cộng tác song song dài hạn.

### Bộ Nhớ, Kỹ Năng Và Prompt

S15 tái sử dụng trực tiếp môi trường bộ nhớ của s09. Trước mỗi lệnh gọi mô hình, nó đọc danh mục `.memory/MEMORY.md`, chọn các bản ghi liên quan đến yêu cầu hiện tại, và truyền nội dung của chúng vào `assemble_system_prompt(context)`. Cuối mỗi lượt, `extract_memories()` giữ lại thông tin có thể giúp ích trong các phiên sau; khi có bản ghi mới được lưu, `consolidate_memories()` sẽ chạy tiếp theo.

Cùng một system prompt đó cũng bao gồm danh tính, hướng dẫn công cụ, không gian làm việc, danh mục kỹ năng và các server MCP đang kết nối. Kỹ năng chỉ đóng góp danh mục rút gọn; hàm `load_skill(name)` sẽ nạp đầy đủ nội dung theo yêu cầu.

### Thu Gọn Và Khôi Phục

Trước khi gọi LLM, S15 chạy đường ống thu gọn ngữ cảnh:

```text
tool_result_budget → snip_compact → micro_compact → compact_history
```

`snip_compact` lưu trữ toàn bộ lịch sử trước khi cắt bớt phần giữa của nó. `micro_compact` chỉ chạy khi vượt quá giới hạn ngữ cảnh: nó lưu các kết quả cũ đã được xử lý trước khi thay thế chúng bằng đường dẫn khôi phục, giữ lại 3 kết quả hoàn chỉnh gần nhất, và dừng lại khi dung lượng đạt khoảng 80% giới hạn. Nếu một kết quả mới xuất hiện quá lớn, S15 sẽ lưu bản xem trước cùng đường dẫn output đầy đủ trước khi xem xét tóm tắt lịch sử.

Lệnh gọi mô hình được bao bọc bởi cơ chế khôi phục lỗi:

- 429: thử lại với cơ chế exponential backoff
- 529: exponential backoff, có thể tùy chọn chuyển sang mô hình dự phòng sau nhiều lần thất bại liên tiếp
- `max_tokens`: tăng mức token tối đa, sau đó yêu cầu viết tiếp
- Prompt quá dài: thu gọn ngữ cảnh phản ứng và thử lại

### Chạy Ngầm Và Cron

Khi một lệnh gọi bash đặt `run_in_background=true`, vòng lặp chính sẽ trả về một kết quả giữ chỗ mà không cần chờ câu lệnh hoàn thành:

```text
should_run_background → start_background_task → placeholder tool_result
background done → task_notification → vòng tiếp theo chèn tin nhắn
```

Chỉ các lệnh gọi bash được đánh dấu rõ ràng mới đi vào nhánh chạy ngầm. Mã thoát khác không hoặc ngoại lệ từ worker sẽ sinh ra thông báo `failed`. Mỗi shell chạy trong nhóm tiến trình riêng, nhóm này sẽ được runtime dừng lại khi câu lệnh hoặc tiến trình Agent kết thúc bình thường hoặc qua tín hiệu `SIGTERM`. Tiến trình nào tự khởi tạo phiên mới có thể thoát khỏi nhóm đó.

Bộ lập lịch cron chạy dưới dạng một daemon thread và kiểm tra mỗi giây một lần. Một công việc chạy một lần bền vững được lưu dưới dạng `pending_delivery` trước khi vào hàng đợi và nằm ở đó cho đến khi lệnh gọi mô hình chứa prompt của nó thành công; một lệnh gọi thất bại sẽ khôi phục công việc đó vào hàng đợi, và việc khởi động lại sẽ đưa nó vào hàng đợi một lần nữa. Do đó, cơ chế chuyển phát đảm bảo ít nhất một lần. CLI quan sát `cron_queue`, hộp thư của Lead, và công việc chạy ngầm ở terminal; bất kỳ sự kiện nào trong số đó đều có thể đánh thức một lượt tương tác tự động của agent.

### Worktree Và MCP

Hành vi worktree theo phạm vi tác vụ kế thừa từ s13 quản lý các thư mục làm việc:

- Một tác vụ đang pending, chưa có người nhận có thể ở lại không gian làm việc chính hoặc được liên kết bằng `create_worktree(name, task_id)` tới một nhánh và thư mục riêng biệt
- Thao tác tạo sẽ xác thực trước tác vụ, tên, đường dẫn, nhánh và Git registry; lệnh Git thất bại sẽ được đối chiếu với registry và trạng thái nhánh, và bất kỳ checkout dở dang nào đều ở trạng thái chưa liên kết và được giữ lại để khôi phục thủ công
- Một teammate rảnh rỗi sẽ nhận nguyên tử một tác vụ đã sẵn sàng; thông tin phân công ghi lại cả `task_id` và `cwd` hiệu dụng của nó
- Lead cũng có thể truyền một `task_id` sẵn sàng vào `spawn_teammate`; luồng chỉ bắt đầu sau khi việc nhận task thành công
- Mọi công cụ file của teammate đều sử dụng `cwd` đó; chỉ teammate sở hữu mới có thể hoàn thành tác vụ, và phân công được giữ nguyên cho đến khi lượt tương tác hiện tại của mô hình kết thúc
- Việc xóa worktree vẫn nằm trong hàm trợ giúp `remove_worktree()` phía host. Mô hình không thể gọi nó. Người dùng hoặc host trước tiên kiểm tra quyền sở hữu tác vụ, phiên phân công, công việc chạy ngầm và trạng thái Git; thao tác xóa mang tính phá hủy cần sự xác nhận riêng của người dùng

Worktree thay đổi thư mục mặc định của các công cụ. Nó phân tách các bản sao làm việc; nó không phải là một hộp cát (sandbox), và việc dọn dẹp nhóm tiến trình không kiểm soát được tiến trình tự mở phiên mới. Đó là lý do tại sao thao tác xóa vẫn thuộc quyền kiểm soát của host.

Việc nhận hoặc giải phóng một Task làm thay đổi phiên bản phân công và vô hiệu hóa phê duyệt kế hoạch cũ. Một lệnh `send_message` thông thường chỉ chuyển phát văn bản; nó không làm thay đổi danh tính Task hay trạng thái kế hoạch.

MCP quản lý năng lực bên ngoài:

- `connect_mcp(name)` kết nối một mock server
- `assemble_tool_pool()` tập hợp các công cụ MCP và từ chối các va chạm tên sau khi chuẩn hóa
- Tên công cụ sử dụng định dạng `mcp__server__tool`

---

## Những Thay Đổi So Với s14

| Phạm vi | s14 MCP | s15 Integrated Harness |
|---|---|---|
| Công cụ tích hợp sẵn | 6 | 25 |
| Công cụ bên ngoài | Các công cụ MCP đang kết nối | Cùng đường dẫn MCP động và chính sách host |
| Cơ chế cục bộ | Công cụ s04, hooks, phân quyền, MCP | Todo, subagent, kỹ năng, thu gọn ngữ cảnh, bộ nhớ, đồ thị tác vụ, bash chạy ngầm, cron, đội ngũ, và worktree |
| Nguồn sự kiện | Đầu vào người dùng và kết quả công cụ | Đầu vào người dùng, kết quả công cụ, prompt từ cron, thông báo chạy ngầm, và sự kiện đội ngũ |

---

## Thử Nghiệm

```sh
cd learn-claude-code
python s15_integrated_harness/code.py
```

Hãy thử:

1. `Inspect this repository and tell me which Python files matter most.`
2. `Search the connected documentation for agent loop guidance.`
3. `Refactor the authentication module and login page in parallel in separate worktrees. Show me each plan before editing.`
4. `Remind me about the meeting in 3 minutes.`
5. `Install the dependencies in the background while you read README.md.`

Những điểm cần quan sát:

- Liệu mỗi lệnh gọi công cụ có đi qua hooks/phân quyền không
- Liệu các công cụ MCP có xuất hiện ở vòng tiếp theo sau khi gọi `connect_mcp` không
- Liệu một lệnh bash với `run_in_background=true` có trả về kết quả giữ chỗ chạy ngầm không
- Liệu cron có tự động nhắc nhở khi đến giờ không
- Liệu các teammate có gửi kế hoạch và tạm dừng trước khi được phê duyệt không
- Liệu một teammate rảnh rỗi có nhận nguyên tử duy nhất một tác vụ sẵn sàng không
- Liệu mọi công cụ file của teammate có chuyển sang `cwd` của tác vụ đã nhận không
- Liệu việc hoàn thành tác vụ có giữ `cwd` đó qua phần còn lại của lượt và chỉ giải phóng khi IDLE không

---

## Bước Tiếp Theo

[s16 Workflow Runtime](../s16_workflow_runtime/) bổ sung một công cụ `Workflow` vào host này. Một quy trình làm việc (workflow) lưu giữ một lộ trình điều phối cố định trong mã nguồn và ghi nhận tiến độ để cùng một lượt chạy có thể tiếp tục lại sau khi bị gián đoạn.

<!-- translation-sync: zh@v14, en@v14, ja@v14 -->
