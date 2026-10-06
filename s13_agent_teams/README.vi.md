# s13: Đội Ngũ Agent — Môi Trường Thực Thi Và Giao Thức Phối Hợp

[English](README.md) · [中文](README.zh.md) · [日本語](README.ja.md) · [Tiếng Việt](README.vi.md)

s01 → ... → [s10](../s10_task_system/) → `s13` → [s14](../s14_mcp_plugin/) → s15 → s16 → s17

> *"Khi một agent không thể ôm trọn toàn bộ công việc, hãy để các đồng đội chia sẻ nhiệm vụ."* — Các đồng đội bền vững, chọn lựa tác vụ dùng chung, worktree tùy chọn và các giao thức phối hợp.
>
> **Lớp Harness**: Đội ngũ (Team) — Cách nhiều agent phân chia công việc, chia sẻ trạng thái và duy trì dưới sự kiểm soát của Lead.

---

## Vấn Đề Đặt Ra

Giả sử chúng ta yêu cầu một agent tái cấu trúc toàn bộ backend. Công việc có thể bao gồm nạp cấu hình, xác thực người dùng, và viết kiểm thử. Một agent duy nhất có thể xử lý tuần tự từng mảng việc đó, nhưng sẽ mất nhiều thời gian hơn và các chi tiết ban đầu sẽ dần trôi khỏi ngữ cảnh của nó.

Đây là trường hợp rất phù hợp để làm việc song song, nhưng người dùng thường chỉ mô tả mục tiêu tổng thể chứ không tự thiết kế đội ngũ agent:

```text
Hãy tái cấu trúc backend mẫu này. Dọn dẹp phần nạp cấu hình,
xác thực, và kiểm thử, bảo toàn các giao diện hiện có,
và đảm bảo toàn bộ kiểm thử đều vượt qua.
```

Harness phải trả lời một loạt các câu hỏi liên kết chặt chẽ với nhau:

1. Ai quyết định việc làm song song là hữu ích, và ai xác nhận việc bổ sung thêm các agent?
2. Làm thế nào để mỗi đồng đội (teammate) duy trì danh tính và ngữ cảnh của mình qua các lần phân công?
3. Kết quả quay trở về Lead bằng cách nào mà không bắt mô hình phải liên tục thăm dò (poll) hộp thư đến?
4. Liệu một đồng đội đang rảnh rỗi có thể tự nhận công việc đã sẵn sàng mà không cần chờ phân công tiếp theo hay không?
5. Tác vụ nên sử dụng thư mục nào khi việc chỉnh sửa song song có thể gây xung đột mã nguồn?
6. Làm sao để việc tắt hệ thống (shutdown) và phê duyệt kế hoạch (plan approval) trở thành các giao thức có thể thực thi và truy vết rõ ràng?

---

## Giải Pháp

![Agent Teams Overview](images/agent-teams-overview.en.svg)

s13 tái sử dụng các công cụ cơ sở, hooks, kiểm tra phân quyền và Hệ thống tác vụ (Task System) của s10, sau đó bổ sung một môi trường thực thi đội ngũ do Lead quản lý:

- **Lead** sở hữu cuộc hội thoại với người dùng, đề xuất phân chia công việc và chờ xác nhận.
- **Các Teammate** chạy các vòng lặp agent độc lập và luân phiên giữa hai trạng thái WORK và IDLE.
- **MessageBus** luân chuyển các tin nhắn thông thường, kết quả và các sự kiện điều khiển thông qua các hộp thư lưu dưới dạng file.
- **Chuyển phát thời gian chạy (Runtime delivery)** tiêu thụ hộp thư của Lead và chèn các sự kiện của nhóm vào lượt tương tác tiếp theo.
- **Bảng tác vụ dùng chung (Shared task board)** cho phép các teammate đang rảnh rỗi tìm kiếm công việc đã sẵn sàng và nhận việc dưới một khóa đồng bộ (lock).
- **Worktree tùy chọn** liên kết một tác vụ với một thư mục làm việc riêng biệt khi công việc cần sự cách ly. Các tác vụ không liên kết sẽ sử dụng thư mục kho chứa mã nguồn bình thường.
- **Các giao thức có kiểu dữ liệu rõ ràng và cổng kế hoạch (plan gate)** giúp trạng thái tắt và phê duyệt trở nên minh bạch, đồng thời chặn các công cụ thay đổi dữ liệu cho đến khi kế hoạch bắt buộc được phê duyệt.

Việc xây dựng đồ thị tác vụ vẫn giữ nguyên cơ chế hai giai đoạn của s10. Lead trước tiên gọi `create_task` cho mọi nút, sau đó sử dụng các ID thời gian chạy được trả về cùng với `update_task(addBlockedBy=...)` trước khi phân công công việc đã sẵn sàng. Chỉ Lead mới có công cụ `update_task`; các teammate có thể liệt kê, nhận và hoàn thành tác vụ nhưng không thể viết lại cấu trúc đồ thị khi nhóm đang vận hành.

Các tác vụ chạy ngầm của s11 và tác vụ theo lịch của s12 không đưa vào chương này. Cả hai cơ chế đó đều không cần thiết cho việc giao tiếp giữa các teammate, nhận tác vụ, hay phê duyệt kế hoạch.

Tất cả những điều này đều thuộc về lớp Harness Đội ngũ (Team). Các teammate không cần một vòng lặp riêng để phát hiện tác vụ, và một worktree không tạo ra một loại agent mới.

---

## Cách Thức Hoạt Động

### 1. Lead Đề Xuất Đội Ngũ Và Chờ Người Dùng Xác Nhận

Khởi động các teammate sẽ làm thay đổi chi phí, mức độ đồng thời và tập hợp các tác nhân có thể chỉnh sửa không gian làm việc. System prompt của Lead làm rõ ranh giới này:

```python
"When parallel work would help, first propose a small team with clear "
"responsibilities and wait for the user's confirmation. Do not call "
"spawn_teammate before the user confirms."
```

Đối với yêu cầu đầu tiên, Lead chỉ đề xuất phương án phân chia:

```text
Tôi đề xuất ba mảng công việc song song:
- config: dọn dẹp phần nạp cấu hình
- auth: tái cấu trúc phần xác thực
- tests: bổ sung độ bao phủ kiểm thử hồi quy

Tôi sẽ khởi động các teammate sau khi bạn xác nhận.
```

Sau khi người dùng đồng ý ("Go ahead"), Lead mới có thể gọi `spawn_teammate`. Lead tạo Task trước và truyền `task_id` ban đầu cho teammate. Người dùng nêu mục tiêu, Lead thiết kế đội ngũ, và người dùng phê duyệt ranh giới thực thi.

### 2. Mỗi Teammate Sở Hữu Một Vòng Lặp Độc Lập

Subagent ở bài s06 là một lệnh gọi một lần (one-shot). Ngược lại, một teammate là một đơn vị thực thi bền vững:

| | s06 Subagent | s13 Teammate |
|---|---|---|
| Vòng đời | Kết thúc sau một lệnh gọi | `WORK → IDLE → WORK` cho đến khi shutdown |
| Ngữ cảnh | Tồn tại cho duy nhất một tác vụ | Duy trì xuyên suốt các lần phân công |
| Giao tiếp | Trả về một kết quả duy nhất | Nhận tin nhắn và phát các sự kiện |
| Phối hợp | Ủy quyền một chiều | Hợp tác hai chiều với Lead |

`TeammateRuntime` cung cấp cho mỗi teammate system prompt, tin nhắn, công cụ và Task hiện tại của riêng nó, sau đó chạy vòng lặp WORK / IDLE trong một daemon thread. Lead có thể tiếp tục điều phối trong khi các teammate làm việc. Các tên `lead` và `agent` được dành riêng cho danh tính thời gian chạy, trong khi `MessageBus` vẫn chấp nhận `lead` làm hộp thư của điều phối viên.

`spawn_teammate` nhận Task ban đầu trước khi luồng bắt đầu. Nếu việc nhận task thất bại, teammate sẽ không được khởi động. Nếu không có Task, các công cụ thao tác workspace và Shell sẽ yêu cầu teammate phải nhận một tác vụ thay vì tự động chuyển về thư mục gốc của kho mã nguồn.

### 3. MessageBus Giữ Việc Giao Tiếp Nằm Ngoài Ngữ Cảnh Mô Hình

Lead và các teammate không thể dùng chung một mảng `messages`. Nếu không, kết quả công cụ của một teammate sẽ rò rỉ vào quá trình suy luận của teammate khác. `MessageBus` cung cấp cho mỗi agent một hộp thư đến `.mailboxes/<name>.jsonl`:

```python
class MessageBus:
    def send(self, from_agent, to_agent, content,
             msg_type="message", metadata=None):
        msg = {
            "from": from_agent,
            "to": to_agent,
            "content": content,
            "type": msg_type,
            "metadata": metadata or {},
        }
        with self._changed:
            MAILBOX_DIR.mkdir(parents=True, exist_ok=True)
            with self._path(to_agent).open("a", encoding="utf-8") as handle:
                handle.write(json.dumps(msg, ensure_ascii=True) + "\n")
            self._changed.notify_all()

    def wait_for_messages(self, agent, timeout=None):
        deadline = None if timeout is None else time.monotonic() + timeout
        with self._changed:
            while not self.peek(agent):
                remaining = (None if deadline is None
                             else deadline - time.monotonic())
                if remaining is not None and remaining <= 0:
                    return []
                self._changed.wait(remaining)
            return self._read_unlocked(agent)
```

Một khóa bảo vệ các file hộp thư khỏi truy cập đồng thời. Đối tượng `Condition` cho phép môi trường chạy đánh thức một teammate khi có tin nhắn và hỗ trợ khoảng thời gian chờ ngắn khi đang ở trạng thái IDLE.

### 4. Môi Trường Chạy Tự Động Chuyển Phát Sự Kiện Hộp Thư

Hàm `read_inbox()` tiêu thụ tin nhắn bằng cách đọc và xóa file hộp thư, do đó Lead duy trì một consumer duy nhất là `consume_lead_inbox()`:

```python
def consume_lead_inbox():
    messages = BUS.read_inbox("lead")
    for message in messages:
        if message["type"].endswith("_response"):
            match_response(...)
    return messages
```

Vòng lặp CLI chờ đầu vào từ terminal và hộp thư của Lead cùng một lúc. Khi có tin nhắn đến, nó tiêu thụ hộp thư trước khi bắt đầu một lượt tương tác mới của Lead:

```text
MessageBus → consume_lead_inbox
           → cập nhật trạng thái giao thức
           → chèn [Team events] vào lịch sử
           → bắt đầu một lượt tương tác mới của Lead
```

Sau khi khởi tạo một teammate, Lead kết thúc lượt hiện tại thay vì liên tục gọi `list_teammates` hay `get_task`. Môi trường chạy sẽ tự động bắt đầu lượt tiếp theo khi có sự kiện nhóm gửi đến.

`check_inbox` không phải là một công cụ của mô hình. Việc tin nhắn đến thuộc quyền quản lý của runtime; mô hình xử lý các sự kiện sau khi runtime đã chuyển phát chúng vào ngữ cảnh của nó.

### 5. Result Và IDLE Là Hai Sự Kiện Tách Biệt

Khi một teammate hoàn thành một phân công, runtime gửi hai sự kiện theo thứ tự:

```text
result:            "Đã tái cấu trúc phần xác thực; các kiểm thử liên quan đều vượt qua."
idle_notification: "Đang chờ thêm công việc."
```

`result` trả lời cho câu hỏi: "Phân công này đã tạo ra kết quả gì?" `idle_notification` trả lời cho câu hỏi: "Liệu teammate này có thể nhận thêm việc không?" Một từ "xong" mơ hồ không thể đại diện cho cả hai dữ kiện trên.

Một teammate đang rảnh rỗi (idle) sẽ không thoát tiến trình. Một tin nhắn trực tiếp hoặc một tác vụ sẵn sàng sẽ đưa nó quay lại trạng thái WORK; một `shutdown_request` sẽ bắt đầu quy trình bắt tay tắt hệ thống một cách an toàn.

### 6. IDLE Kiểm Tra Hộp Thư Trước Khi Tìm Tác Vụ Sẵn Sàng

Khi ở trạng thái IDLE, teammate ưu tiên xử lý tin nhắn trước, sau đó mới kiểm tra bảng tác vụ dùng chung:

```python
while True:
    inbox = BUS.wait_for_messages(name, IDLE_SCAN_INTERVAL)
    if inbox:
        should_stop = handle_messages(inbox)
        if should_stop or messages[-1]["role"] == "user":
            break
        continue

    task = claim_next_task(name)
    if task:
        messages.append({
            "role": "user",
            "content": f"[Auto-claimed task {task.id}] {task.subject}",
        })
        break
```

Yêu cầu dừng (shutdown), phê duyệt kế hoạch, và các chỉ thị trực tiếp từ Lead cần được xử lý trước công việc tự phát. Nếu không có tin nhắn và không có tác vụ nào sẵn sàng, teammate vẫn ở trạng thái IDLE. Một tác vụ bị chặn có thể trở nên sẵn sàng sau khi một teammate khác hoàn thành điều kiện tiên quyết của nó.

### 7. Phát Hiện Và Nhận Việc Là Hai Bước Riêng Biệt, Và Nhận Việc Là Nguyên Tử (Atomic)

Quét tác vụ chỉ tìm kiếm các ứng viên:

```python
def scan_unclaimed_tasks() -> list[Task]:
    return [
        task for task in list_tasks()
        if task.status == "pending"
        and task.owner is None
        and can_start(task.id)
    ]
```

Danh sách này chỉ là ảnh chụp tức thời (snapshot). Một teammate khác, hoặc một tiến trình harness khác dùng chung thư mục tác vụ, có thể thấy cùng một tác vụ đó. Do đó, việc thay đổi quyền sở hữu diễn ra bên trong `claim_task()` dưới sự bảo vệ của `task_store_lock()`, kết hợp khóa trong tiến trình với khóa file:

```python
def claim_task(task_id: str, owner: str) -> str:
    with task_store_lock():
        task = load_task(task_id)
        if task.status != "pending" or task.owner is not None:
            return "Task is no longer available"
        if _owner_in_progress(owner):
            return "Owner must complete its current task first"
        if not can_start(task_id):
            return "Task is blocked"
        cwd, error = task_worktree_cwd(task)
        if error:
            return f"Cannot claim {task_id}: {error}"
        task.owner = owner
        task.status = "in_progress"
        save_task(task)
        teammate_assignments[owner] = {"task_id": task.id, "cwd": cwd}
        return f"Claimed {task.id}"
```

Nhiều teammate có thể cùng phát hiện ra một ứng viên, nhưng chỉ có một lệnh nhận việc duy nhất có thể chuyển trạng thái của nó sang `in_progress`. Các file tác vụ được ghi qua file tạm và thay thế nguyên tử trong khi vẫn giữ khóa lưu trữ. Một teammate cũng phải hoàn thành tác vụ hiện tại trước khi nhận tác vụ khác, và một liên kết worktree hỏng sẽ từ chối thực thi thay vì tự động chuyển về thư mục kho mã nguồn.

### 8. Công Việc Đã Nhận Tái Sử Dụng Cùng Một Vòng Lặp WORK

Sau khi nhận tác vụ thành công, runtime sẽ đưa ID, tiêu đề và mô tả của tác vụ vào danh sách messages của teammate:

```text
tác vụ sẵn sàng xuất hiện
  → teammate IDLE phát hiện ra nó
  → claim_task ghi owner và in_progress
  → tác vụ được đưa vào messages của teammate
  → WORK
  → complete_task
  → result + idle_notification
  → IDLE
```

Teammate sử dụng cùng một cơ chế gọi mô hình, công cụ file, Shell, cổng kế hoạch, báo cáo kết quả và giao thức tắt tương tự như khi Lead trực tiếp phân công. Việc tự phát hiện tác vụ chỉ là một điểm vào khác của vòng lặp WORK hiện có.

### 9. Tác Vụ Lựa Chọn Thư Mục Làm Việc Của Công Cụ

Trường `Task.worktree` là tùy chọn:

```python
@dataclass
class Task:
    id: str
    subject: str
    description: str
    status: str
    owner: str | None
    blockedBy: list[str]
    worktree: str | None = None
```

Lead có thể tạo và liên kết một worktree khi việc phân chia thư mục là cần thiết:

```python
create_worktree(name="auth-refactor", task_id="task_1a2b3c4d")
```

`create_worktree` là công cụ chỉ dành riêng cho Lead. Nó tiếp nhận một tác vụ đang pending, chưa có chủ và chưa được liên kết, xác thực tên, đường dẫn, nhánh và Git registry, tạo thư mục checkout, sau đó ghi liên kết vào tác vụ. Nếu Git báo lỗi sau khi đã tạo nhánh hoặc checkout, runtime sẽ báo cáo một thao tác dở dang, để tác vụ ở trạng thái chưa liên kết, và giữ nguyên các thành phần đó để khôi phục thủ công. Các teammate chỉ nhìn thấy các công cụ tác vụ và file.

Việc nhận tác vụ sẽ lưu đường dẫn thư mục đã phân giải vào `teammate_assignments`; các lớp bọc của `bash`, `read_file`, `write_file`, `edit_file`, và `glob` của teammate đó sẽ đọc thư mục từ thông tin phân công. Một tác vụ không có worktree sẽ phân giải về `WORKDIR`; một teammate chưa nhận Task nào sẽ không thể sử dụng các công cụ thao tác workspace này:

```python
cwd, error = task_worktree_cwd(task)
if not error:
    teammate_assignments[owner] = {
        "task_id": task.id,
        "cwd": cwd,
    }
```

`complete_task(task_id, owner)` kiểm tra xem người gọi có đúng là chủ sở hữu của tác vụ đang thực hiện hay không. Việc hoàn thành thành công sẽ ghi lại kết quả nhưng vẫn giữ nguyên thư mục làm việc đã phân công cho đến khi lượt tương tác hiện tại của mô hình kết thúc. Điều này cho phép các lệnh gọi công cụ tiếp theo trong cùng phản hồi vẫn nằm trong worktree của tác vụ. Runtime sẽ giải phóng phân công khi teammate quay về trạng thái IDLE; nếu việc hoàn thành thất bại, phân công vẫn được giữ để teammate có thể sửa chữa tác vụ và thử lại.

Sau khi khởi động lại, `assignment_cwd()` có thể xây dựng lại phân công đang thực hiện từ thông tin chủ sở hữu tác vụ và liên kết worktree bền vững. Nó cũng thay thế một phiên thuê (lease) cục bộ đã lỗi thời khi cùng một chủ sở hữu đã chuyển sang tác vụ khác. Một liên kết bị thiếu hoặc không hợp lệ sẽ từ chối hoạt động thay vì âm thầm điều hướng công việc về thư mục kho mã nguồn.

> Worktree phân tách các thư mục làm việc và các nhánh Git. Nó không phải là một môi trường hộp cát (sandbox): các lệnh Shell vẫn có thể truy cập các đường dẫn và tài nguyên mà tiến trình cha có quyền truy cập.

### 10. Việc Xóa Worktree Thuộc Quyền Của Host

Mô hình có thể tạo một worktree liên kết với tác vụ, nhưng không thể tự xóa nó. Việc dọn dẹp vẫn là một hàm trợ giúp của host để người dùng hoặc host có thể kiểm tra quyền sở hữu tác vụ, phiên phân công, và trạng thái Git trước. Trợ giúp này từ chối xóa các liên kết tác vụ đang pending hoặc in-progress cũng như các phiên phân công ở lượt hiện tại. Nếu không có lựa chọn phá hủy rõ ràng, các file đã theo dõi (tracked), chưa theo dõi (untracked) và bị bỏ qua (ignored) đều sẽ chặn thao tác xóa.

`remove_worktree(name, discard_changes=True)` được dành riêng cho mã nguồn phía host đã có xác nhận rõ ràng của người dùng. Cả hai nhánh xóa đều giữ lại nhánh `wt/<name>`, bao gồm cả các commit cục bộ chưa đẩy lên remote. Thao tác xóa thành công sẽ xóa bỏ liên kết của tác vụ vì checkout không còn tồn tại nữa.

```text
worktree sạch           → host có thể xóa thư mục và giữ lại nhánh wt/<name>
worktree có thay đổi    → người dùng quyết định giữ lại hay hủy bỏ
tác vụ pending/running  → từ chối xóa
```

Việc hoàn thành tác vụ cũng tách biệt hoàn toàn với việc dọn dẹp worktree. `complete_task` ghi lại kết quả tác vụ; sau khi teammate chuyển sang trạng thái IDLE, người dùng hoặc host có thể kiểm tra, hợp nhất, giữ lại hoặc xóa worktree.

### 11. Các Tin Nhắn Điều Khiển Sử Dụng Kiểu Dữ Liệu Và Mã Request ID

Văn bản tự do hoạt động tốt cho việc cộng tác thông thường, nhưng thao tác tắt hệ thống và phê duyệt không nên dựa vào việc phỏng đoán ý định. Chúng sử dụng các thông điệp có cấu trúc:

![Team Protocols](images/team-protocols-overview.en.svg)

```python
@dataclass
class ProtocolState:
    request_id: str
    type: str
    sender: str
    target: str
    status: str
    payload: str
    work_version: int | None = None
    task_id: str | None = None


pending_requests: dict[str, ProtocolState] = {}
```

Quy trình tắt hệ thống (shutdown) diễn ra như sau:

```text
Lead tạo một yêu cầu tắt hệ thống đang chờ xử lý
  → shutdown_request(request_id) được gửi vào hộp thư của teammate
  → teammate hoàn thành bước hiện tại
  → shutdown_response(request_id) trả về cho Lead
  → request_id định vị yêu cầu ban đầu
  → trạng thái pending chuyển thành approved và vòng lặp của teammate kết thúc
```

ID giúp liên kết một phản hồi với đúng yêu cầu tương ứng, kiểu thông điệp (type) ngăn việc một phản hồi không khớp làm thay đổi trạng thái, và trạng thái (status) ngăn các phản hồi trùng lặp bị áp dụng hai lần.

### 12. Phê Duyệt Kế Hoạch Ràng Buộc Việc Thực Thi

Giao thức kế hoạch hoạt động theo chiều ngược lại:

```text
Lead → plan_request
teammate → plan_approval_request(request_id, plan)
Lead → plan_approval_response(request_id, approve, feedback)
```

Khi Lead biết trước rằng một teammate bắt buộc phải lập kế hoạch trước, `spawn_teammate(..., task_id=task.id, require_plan=True)` sẽ nhận Task và kích hoạt cổng kế hoạch (gate) trước khi luồng của teammate bắt đầu. Hàm `request_plan` cũng có thể yêu cầu kế hoạch từ một teammate đang chạy.

Bộ điều phối công cụ (Tool dispatch) thực thi cổng kiểm soát này:

```python
def _run_teammate_tool(name, block, handlers):
    gate = plan_gates.get(name, "not_required")
    if block.name in {"bash", "write_file", "edit_file"} and gate not in {
        "not_required", "approved"
    }:
        return f"Blocked: plan status is {gate}."
    try:
        return handlers[block.name](**block.input)
    except Exception as error:
        return f"Error: {type(error).__name__}: {error}"
```

Trong khi trạng thái là `required`, `pending`, hoặc `rejected`, teammate có thể đọc file và gửi hoặc sửa đổi kế hoạch, nhưng không thể chạy các lệnh Shell, ghi file, hoặc sửa file. Một kế hoạch được gửi lên sẽ ghi nhận tác vụ hiện tại và phiên bản công việc (work version) của teammate. Việc nhận hoặc giải phóng một Task sẽ thay đổi phiên bản đó và vô hiệu hóa phê duyệt cũ; một tin nhắn thông thường không làm thay đổi danh tính tác vụ hay trạng thái phê duyệt.

Các teammate không đọc đầu vào của người dùng từ các luồng chạy ngầm của chúng. Một câu lệnh nguy hiểm hoặc một đường dẫn nằm ngoài không gian làm việc sẽ trả về lỗi phân quyền để Lead có thể xử lý quyết định đó cùng với người dùng.

---

## Một Lượt Chạy Hoàn Chỉnh

```text
s13 >> Đưa việc tái cấu trúc backend lên bảng tác vụ dùng chung. Dọn dẹp
       cấu hình, xác thực, và kiểm thử song song ở những chỗ có thể.
       Dùng worktree cho phần xác thực, bảo toàn các giao diện hiện có,
       và đảm bảo kiểm thử vượt qua.

Lead: Tôi đề xuất config, auth, và tests làm ba mảng việc.
      Tôi có thể bắt đầu đội ngũ được chưa?

s13 >> Đồng ý (Go ahead).

[task] config created
[task] auth created → worktree auth-refactor
[task] tests created
[claim] alice → config (cwd: repository)
[claim] bob → auth (cwd: .worktrees/auth-refactor)
[teammate] alice spawned
[teammate] bob spawned
[complete] auth
[bus] bob → lead (result) ...
[bus] bob → lead (idle_notification) ...
[wake: 2 sự kiện nhóm → lượt mới]
Lead: Tôi đã nhận kết quả xác thực và sẽ điều phối phần còn lại.
```

Terminal hiển thị yêu cầu của người dùng, đề xuất của Lead, trạng thái tác vụ, các lượt nhận việc, các thư mục đã chọn, kết quả, các chuyển đổi trạng thái IDLE, và các sự kiện điều khiển. Người dùng không cần phải đặt tên cho Lead hay yêu cầu nó phải kiểm tra hộp thư.

---

## Những Thay Đổi So Với s10

| Thành phần | s10 | s13 |
|---|---|---|
| Số lượng Agent | Một agent | Một Lead cùng các teammate bền vững |
| Luồng người dùng | Thực thi yêu cầu | Đề xuất đội ngũ, sau đó xác nhận khởi động |
| Giao tiếp | Không có | Hộp thư file cùng cơ chế chuyển phát runtime |
| Vòng đời | Một vòng lặp | Teammate `WORK / IDLE / shutdown` |
| Chia sẻ công việc | Một agent dùng các công cụ tác vụ | Quét IDLE cùng các lượt nhận tác vụ nguyên tử |
| Thư mục làm việc | `WORKDIR` của kho mã nguồn | Một Task đã nhận, kèm worktree tùy chọn |
| Báo cáo | Đầu ra của agent hiện tại | Phân tách `result` và `idle_notification` |
| Điều khiển | Không có | Các giao thức shutdown và duyệt kế hoạch có kiểu rõ ràng |
| Thực thi | Không có ràng buộc đội ngũ | Kế hoạch bắt buộc chặn các công cụ thay đổi dữ liệu |

---

## Thử Nghiệm

```sh
cd learn-claude-code
python s13_agent_teams/code.py
```

Bắt đầu bằng một yêu cầu thông thường:

```text
Put the backend refactor on a shared task board. Complete configuration,
authentication, and tests in parallel where dependencies allow. Use a
worktree for authentication, preserve existing interfaces, and summarize
the result.
```

Sau khi Lead đề xuất đội ngũ, phản hồi:

```text
Go ahead.
```

Quan sát thư mục `.tasks/` chuyển từ `pending` sang `in_progress` và `completed`, `.mailboxes/` chuyển phát `result` và `idle_notification`, và `.worktrees/` chỉ xuất hiện cho tác vụ được liên kết. Đồng thời kiểm tra xem các tin nhắn trực tiếp có được ưu tiên hơn việc quét bảng tác vụ hay không, và việc gọi `complete_task` thất bại có không làm đặt lại thư mục làm việc của teammate hay không.

---

## Bước Tiếp Theo

Lead và các teammate chỉ có thể gọi các công cụ được định nghĩa trực tiếp trong `code.py`. Việc kết nối Jira, một nền tảng triển khai, hay một kho tri thức vẫn đòi hỏi các schema công cụ và hàm xử lý riêng cho từng hệ thống bên ngoài. Bất kỳ thay đổi nào đối với các công cụ bên ngoài đó cũng đòi hỏi phải sửa đổi mã nguồn của khóa học.

s14 MCP Tools → Kết nối các dịch vụ bên ngoài trong thời gian chạy thông qua một giao thức khám phá và triệu gọi duy nhất, sau đó bổ sung các công cụ của chúng vào kho công cụ chung.

<!-- translation-sync: zh@v12, en@v12, ja@v12 -->
