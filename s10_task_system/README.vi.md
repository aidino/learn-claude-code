# s10: Hệ Thống Tác Vụ — Từ Danh Sách Kiểm Tra Thực Thi Đến Trạng Thái Tác Vụ Phối Hợp

[English](README.md) · [中文](README.zh.md) · [日本語](README.ja.md) · [Tiếng Việt](README.vi.md)

s01 → ... → s08 → s09 → `s10` → [s11](../s11_background_tasks/) → s12 → ... → s16 → s17

> *"Chia nhỏ mục tiêu lớn thành các tác vụ nhỏ, sắp xếp thứ tự, lưu trữ bền vững"* — Đồ thị tác vụ lưu bền vững dạng file, nền tảng cho sự cộng tác đa agent.
>
> **Lớp Harness**: Tác vụ (Tasks) — Mục tiêu bền vững, khôi phục tiến độ dễ dàng.

---

## Vấn Đề Đặt Ra

Công cụ TodoWrite ở bài s05 cho phép một agent ghi lại các bước của tác vụ hiện tại. Mỗi mục trong danh sách kiểm tra (checklist) đều có nội dung và trạng thái, giúp agent theo dõi những gì còn lại cần làm.

Khi một dự án được chia thành ba tác vụ—tạo các bảng cơ sở dữ liệu, viết API, và viết kiểm thử—Harness cũng cần biết mối quan hệ giữa chúng: API phải chờ các bảng cơ sở dữ liệu, và kiểm thử phải chờ một API ổn định. Hệ thống cũng cần ghi nhận ai chịu trách nhiệm cho từng tác vụ.

TodoWrite không ghi lại các mối quan hệ phụ thuộc hoặc sự phân công này. Nó có thể cho thấy "viết API" chưa hoàn thành, nhưng Harness không thể dùng thông tin đó để quyết định liệu tác vụ đã sẵn sàng để bắt đầu hay chưa.

Chương này bổ sung một Hệ Thống Tác Vụ (Task System). Mỗi tác vụ có ID và trạng thái riêng; `blockedBy` ghi lại các tác vụ tiên quyết, và `owner` ghi lại agent chịu trách nhiệm cho tác vụ đó.

---

## Giải Pháp

![Task System Overview](images/task-system-overview.en.svg)

Mã nguồn giữ nguyên năm công cụ cơ sở của s04, Cổng phân quyền (Permission), Hooks, và hàm `execute_tool` dùng chung, sau đó bổ sung 6 công cụ tác vụ, cơ chế lưu trữ bền vững trong thư mục `.tasks/`, cùng các kiểm tra phụ thuộc `blockedBy`.

So sánh TodoWrite và Hệ Thống Tác Vụ:

| | TodoWrite (s05) | Task System (s10) |
|---|---|---|
| Vai trò | Danh sách kiểm tra thực thi cho tác vụ hiện tại | Hệ thống tác vụ có khả năng khôi phục |
| Lưu trữ | Trạng thái trong bộ nhớ tiến trình / phiên | `.tasks/{id}.json` |
| Phụ thuộc | Không có | Đồ thị phụ thuộc `blockedBy` |
| Vòng đời | Phiên hiện tại / tác vụ hiện tại | Xuyên suốt các phiên (Cross-session) |
| Phối hợp | Không hỗ trợ nhận tác vụ | `owner` / nhận tác vụ (claim) |
| Trạng thái | pending / in_progress / completed | pending / in_progress / completed |
| Độ chi tiết | Các bước riêng của agent | Các tác vụ có thể nhận, theo dõi và gỡ chặn |
| Cơ chế cập nhật | Thay thế toàn bộ danh sách kiểm tra | Tạo/lấy/cập nhật/liệt kê từng bản ghi riêng lẻ |

---

## Cách Thức Hoạt Động

![Task DAG](images/task-dag.en.svg)

### Task: Cấu Trúc Dữ Liệu

Mỗi tác vụ là một file JSON, được lưu trữ trong thư mục `.tasks/`:

```python
@dataclass
class Task:
    id: str
    subject: str
    description: str
    status: str          # pending | in_progress | completed
    owner: str | None    # Agent chịu trách nhiệm cho tác vụ này
    blockedBy: list[str] # Danh sách ID của các tác vụ phụ thuộc
```

Các ID sử dụng tiền tố `task_` theo sau bởi 8 ký tự thập lục phân ngẫu nhiên. Các file được tạo độc quyền; một ID đã tồn tại sẽ bị loại bỏ và tạo lại mã mới.

`TaskStore` xác thực các ID tác vụ và đọc/ghi các file JSON. `TASKS = TaskStore(TASKS_DIR)` là kho lưu trữ được sử dụng trong chương này.

### create_task: Tạo Tác Vụ

```python
def create_task(subject: str, description: str = "") -> Task:
    return TASKS.create(subject, description)
```

`TaskStore.create` kiểm tra tiêu đề (subject), cấp phát một ID ngẫu nhiên, và ghi file `.tasks/{id}.json`. Một tác vụ mới luôn bắt đầu với danh sách `blockedBy` rỗng. Kết quả công cụ sẽ trả về ID được tạo trong thời gian chạy cho mô hình.

### update_task: Thêm Phụ Thuộc Bằng Các ID Đã Trả Về

```python
def update_task(task_id: str, addBlockedBy: list[str]) -> Task:
    return TASKS.update_dependencies(task_id, addBlockedBy)
```

Việc xây dựng đồ thị tác vụ sử dụng hai giai đoạn: tạo tất cả các nút trước, sau đó gọi `update_task` với các ID do `create_task` trả về để thêm các cạnh phụ thuộc. Điều này rất quan trọng khi mô hình phát ra nhiều lệnh gọi công cụ trong cùng một phản hồi: các lệnh gọi đồng cấp được hình thành trước khi có bất kỳ kết quả công cụ nào tồn tại, vì vậy một lệnh gọi `create_task` không thể sử dụng ID vừa được tạo của một lệnh gọi khác cùng lượt.

`update_task` xác thực toàn bộ thay đổi trước khi lưu. Mục tiêu và các phụ thuộc phải tồn tại, mục tiêu phải đang ở trạng thái pending và chưa có người nhận (unowned), và các cạnh mới không được tạo ra quan hệ tự phụ thuộc hoặc vòng lặp phụ thuộc (cycles). Việc lặp lại một cạnh đã tồn tại là an toàn và không gây trùng lặp.

### can_start: Kiểm Tra Phụ Thuộc

Một tác vụ chỉ có thể bắt đầu sau khi tất cả các phụ thuộc trong `blockedBy` của nó đã **hoàn thành** (`completed`):

```python
def can_start(task_id: str) -> bool:
    return not incomplete_dependencies(load_task(task_id))
```

`incomplete_dependencies` nạp từng tác vụ tiên quyết. Một tác vụ không thể được nhận (claim) nếu bất kỳ điều kiện tiên quyết nào chưa hoàn thành hoặc file của nó không còn tồn tại.

### claim_task: Nhận Tác Vụ

Khi agent bắt đầu thực hiện một tác vụ, nó gọi `claim_task`: thiết lập `owner`, chuyển trạng thái từ `pending` → `in_progress`. Trường `owner` ghi lại ai đã nhận tác vụ:

```python
def claim_task(task_id: str, owner: str = "agent") -> str:
    task = load_task(task_id)
    if task.status != "pending":
        return f"Task {task_id} is {task.status}, cannot claim"
    dependencies = incomplete_dependencies(task)
    if dependencies:
        return f"Blocked by: {dependencies}"
    task.owner = owner
    task.status = "in_progress"
    TASKS.save(task)
    return f"Claimed {task_id} ({task.subject})"
```

Yêu cầu nhận tác vụ sẽ bị từ chối nếu tác vụ không ở trạng thái pending hoặc các phụ thuộc của nó chưa hoàn thành. S10 chỉ cập nhật trạng thái tác vụ một cách tuần tự.

### complete_task: Hoàn Thành Và Gỡ Chặn

Khi một tác vụ hoàn tất, chuyển trạng thái của nó sang `completed`. Đồng thời quét tất cả các tác vụ khác để tìm các tác vụ hạ nguồn **vừa mới được gỡ chặn**:

```python
def complete_task(task_id: str, owner: str = "agent") -> str:
    task = load_task(task_id)
    if task.status != "in_progress":
        return f"Task {task_id} is {task.status}, cannot complete"
    if task.owner != owner:
        return f"Task {task_id} is owned by {task.owner}, not {owner}"
    ready_before = {t.id for t in list_tasks()
                    if t.status == "pending" and t.blockedBy
                    and can_start(t.id)}
    task.status = "completed"
    TASKS.save(task)
    unblocked = [t.subject for t in list_tasks()
                 if t.status == "pending" and t.blockedBy
                 and t.id not in ready_before
                 and can_start(t.id)]
    msg = f"Completed {task_id} ({task.subject})"
    if unblocked:
        msg += f"\nUnblocked: {', '.join(unblocked)}"
    return msg
```

Sau khi hoàn thành "schema", hàm `can_start` sẽ trả về True cho "endpoints" và "docs"; chúng có thể bắt đầu được thực hiện.

### get_task: Xem Chi Tiết Đầy Đủ

`list_tasks` chỉ hiển thị bản tóm tắt một dòng. `get_task` trả về toàn bộ JSON của tác vụ, bao gồm phần mô tả và chi tiết phụ thuộc. Khi khôi phục qua các phiên làm việc, agent cần đọc mô tả đầy đủ để tiếp tục công việc:

```python
def get_task(task_id: str) -> str:
    task = load_task(task_id)
    return json.dumps(asdict(task), indent=2)
```

### Máy Trạng Thái: Hai Hành Động, Ba Trạng Thái

```
pending ──claim──→ in_progress ──complete──→ completed
```

Ở đây `claim` / `complete` là các hành động, trong khi `pending` / `in_progress` / `completed` là các trạng thái:

- **claim_task**: `pending` → `in_progress`. Gán owner, bắt đầu làm việc.
- **complete_task**: `in_progress` → `completed`. Đánh dấu tác vụ đã xong và gỡ chặn các tác vụ phía sau.

### Kết Hợp Toàn Bộ Quy Trình

```python
# Giai đoạn 1: tạo từng nút và nhận ID thời gian chạy
schema = create_task("setup database schema")
endpoints = create_task("create API endpoints")
tests = create_task("write tests")
docs = create_task("write docs")

# Giai đoạn 2: thêm các cạnh phụ thuộc bằng các ID vừa nhận
update_task(endpoints.id, addBlockedBy=[schema.id])
update_task(tests.id, addBlockedBy=[endpoints.id])
update_task(docs.id, addBlockedBy=[schema.id])

# Agent nhận tác vụ khả dụng đầu tiên
claim_task(schema.id)       # ✓ Đã nhận (không có phụ thuộc)
complete_task(schema.id)    # ✓ Hoàn thành → gỡ chặn endpoints, docs

claim_task(endpoints.id)    # ✓ Đã nhận (schema đã hoàn thành)
complete_task(endpoints.id) # ✓ Hoàn thành → gỡ chặn tests

claim_task(docs.id)         # ✓ Đã nhận (schema đã hoàn thành)
complete_task(docs.id)      # ✓ Hoàn thành

claim_task(tests.id)        # ✓ Đã nhận (endpoints đã hoàn thành)
complete_task(tests.id)     # ✓ Hoàn thành
```

Mỗi lệnh `create_task` ghi một file JSON; `update_task`, `claim_task`, và `complete_task` sẽ cập nhật file đó. Xuyên suốt các phiên làm việc, thư mục `.tasks/` vẫn tồn tại bền vững — agent đọc các file này để khôi phục tiến độ công việc.

---

## Thử Nghiệm

```sh
cd learn-claude-code
python s10_task_system/code.py
```

Hãy thử các câu lệnh prompt sau:

1. `Create tasks: setup database schema, create API endpoints (depends on schema), write tests (depends on endpoints), write docs (depends on schema)`
2. `List all tasks and their statuses`
3. `Claim the first unblocked task and complete it`
4. `List tasks again — which ones are now unblocked?`

Những điểm cần quan sát: Các file JSON có được tạo trong thư mục `.tasks/` không? Sau khi hoàn thành một tác vụ, các tác vụ bị chặn có được giải phóng (unblocked) không?

---

## Bước Tiếp Theo

Đồ thị tác vụ đã sẵn sàng, nhưng việc chạy toàn bộ bộ kiểm thử, cài đặt gói phụ thuộc, hay các lệnh triển khai có thể mất nhiều thời gian. Khi các lệnh này chạy đồng bộ, Vòng lặp Agent sẽ bị chặn hoàn toàn ở lệnh gọi công cụ hiện tại và không thể tiếp tục cho đến khi lệnh đó chạy xong.

s11 Background Tasks → Các thao tác chậm chạy dưới nền. Vòng lặp Agent có thể tiếp tục xử lý các tác vụ khác và nhận thông báo khi tác vụ chạy ngầm hoàn tất.

<!-- translation-sync: zh@v5, en@v5, ja@v5 -->
