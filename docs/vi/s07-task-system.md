# s07: Hệ Thống Tác Vụ (Task System)

`s01 > s02 > s03 > s04 > s05 > s06 | [ s07 ] > s08 > s09 > s10 > s11 > s12`

> *"Break big goals into small tasks, order them, persist to disk"* -- chia nhỏ mục tiêu lớn thành các tác vụ nhỏ, sắp xếp thứ tự, lưu trữ bền vững trên ổ đĩa. Một đồ thị tác vụ dạng file kèm theo các phụ thuộc là nền tảng cốt lõi cho sự cộng tác đa agent.
>
> **Tầng Harness**: Tác vụ bền vững (Persistent tasks) -- những mục tiêu tồn tại độc lập, dài hạn hơn bất kỳ phiên hội thoại đơn lẻ nào.

## Vấn đề

`TodoManager` ở bài s03 chỉ là một danh sách kiểm tra phẳng nằm trong bộ nhớ RAM: không có quan hệ thứ tự, không có quan hệ phụ thuộc (dependencies), và trạng thái chỉ dừng lại ở mức xong-hay-chưa. Trong thực tế, các mục tiêu luôn có cấu trúc phức tạp -- tác vụ B phụ thuộc vào tác vụ A, tác vụ C và D có thể chạy song song, tác vụ E phải đợi cả C và D hoàn thành.

Nếu không có các mối quan hệ rõ ràng, agent không thể xác định việc nào đã sẵn sàng, việc nào đang bị chặn, hay việc nào có thể chạy đồng thời. Và bởi vì danh sách này chỉ nằm trong bộ nhớ tạm, cơ chế thu gọn ngữ cảnh (s06) sẽ xóa sạch nó.

## Giải pháp

Nâng cấp danh sách kiểm tra thành một **đồ thị tác vụ** (task graph) được lưu trữ bền vững trên ổ đĩa. Mỗi tác vụ là một file JSON chứa trạng thái và các phụ thuộc (`blockedBy`). Đồ thị này trả lời ba câu hỏi quan trọng tại bất kỳ thời điểm nào:

- **Việc gì đã sẵn sàng làm?** -- các tác vụ ở trạng thái `pending` và có danh sách `blockedBy` rỗng.
- **Việc gì đang bị chặn?** -- các tác vụ đang phải chờ những phụ thuộc chưa hoàn thành.
- **Việc gì đã hoàn thành?** -- các tác vụ `completed`, khi hoàn thành sẽ tự động gỡ chặn cho các tác vụ phụ thuộc tiếp theo.

```
.tasks/
  task_1.json  {"id":1, "status":"completed"}
  task_2.json  {"id":2, "blockedBy":[1], "status":"pending"}
  task_3.json  {"id":3, "blockedBy":[1], "status":"pending"}
  task_4.json  {"id":4, "blockedBy":[2,3], "status":"pending"}

Đồ thị tác vụ (DAG):
                 +----------+
            +--> | task 2   | --+
            |    | pending  |   |
+----------+     +----------+    +--> +----------+
| task 1   |                          | task 4   |
| completed| --> +----------+    +--> | blocked  |
+----------+     | task 3   | --+     +----------+
                 | pending  |
                 +----------+

Thứ tự:       task 1 phải hoàn thành trước task 2 và 3
Song song:    task 2 và 3 có thể thực thi cùng một lúc
Phụ thuộc:    task 4 đợi cả task 2 và task 3 hoàn thành
Trạng thái:   pending -> in_progress -> completed
```

Đồ thị tác vụ này trở thành xương sống điều phối cho toàn bộ các bài viết từ s07 trở đi: thực thi tác vụ chạy ngầm (s08), đội ngũ đa agent (s09+), và cô lập git worktree (s12) đều đọc và ghi trên cùng một cấu trúc này.

## Cách thức hoạt động

1. **TaskManager**: mỗi tác vụ tương ứng một file JSON, hỗ trợ đầy đủ thao tác CRUD cùng đồ thị quan hệ phụ thuộc.

```python
class TaskManager:
    def __init__(self, tasks_dir: Path):
        self.dir = tasks_dir
        self.dir.mkdir(exist_ok=True)
        self._next_id = self._max_id() + 1

    def create(self, subject, description=""):
        task = {"id": self._next_id, "subject": subject,
                "status": "pending", "blockedBy": [],
                "owner": ""}
        self._save(task)
        self._next_id += 1
        return json.dumps(task, indent=2)
```

2. **Giải quyết phụ thuộc (Dependency resolution)**: khi một tác vụ chuyển sang trạng thái hoàn thành, ID của nó sẽ được tự động xóa khỏi danh sách `blockedBy` của tất cả các tác vụ khác, từ đó tự động giải phóng các tác vụ bị chặn.

```python
def _clear_dependency(self, completed_id):
    for f in self.dir.glob("task_*.json"):
        task = json.loads(f.read_text(encoding="utf-8"))
        if completed_id in task.get("blockedBy", []):
            task["blockedBy"].remove(completed_id)
            self._save(task)
```

3. **Cập nhật trạng thái và liên kết phụ thuộc**: hàm `update` xử lý việc chuyển trạng thái và thêm/bớt các cạnh phụ thuộc trong đồ thị.

```python
def update(self, task_id, status=None,
           add_blocked_by=None, remove_blocked_by=None):
    task = self._load(task_id)
    if status:
        task["status"] = status
        if status == "completed":
            self._clear_dependency(task_id)
    if add_blocked_by:
        task["blockedBy"] = list(set(task["blockedBy"] + add_blocked_by))
    if remove_blocked_by:
        task["blockedBy"] = [x for x in task["blockedBy"] if x not in remove_blocked_by]
    self._save(task)
```

4. Bốn công cụ quản lý tác vụ được đăng ký vào bảng điều phối:

```python
TOOL_HANDLERS = {
    # ...base tools...
    "task_create": lambda **kw: TASKS.create(kw["subject"]),
    "task_update": lambda **kw: TASKS.update(kw["task_id"], kw.get("status")),
    "task_list":   lambda **kw: TASKS.list_all(),
    "task_get":    lambda **kw: TASKS.get(kw["task_id"]),
}
```

Kể từ s07, đồ thị tác vụ là cơ chế mặc định cho công việc nhiều bước. Công cụ Todo ở s03 vẫn được giữ lại cho các danh sách kiểm tra nhanh gọn trong một phiên làm việc đơn lẻ.

## Những điểm thay đổi so với s06

| Thành phần         | Trước đây (s06)              | Sau khi cập nhật (s07)                      |
|--------------------|------------------------------|---------------------------------------------|
| Công cụ            | 5                            | 8 (`task_create/update/list/get`)           |
| Mô hình kế hoạch   | Danh sách phẳng (trong RAM)  | Đồ thị tác vụ kèm quan hệ phụ thuộc (ổ đĩa) |
| Quan hệ công việc  | Không có                     | Các cạnh `blockedBy`                        |
| Theo dõi trạng thái| Hoàn thành hay chưa          | `pending` -> `in_progress` -> `completed`   |
| Lưu trữ bền vững   | Bị mất khi thu gọn ngữ cảnh  | Tồn tại qua các lần thu gọn và khởi động lại|

## Trải nghiệm thực tế

```sh
cd learn-claude-code
python agents/s07_task_system.py
```

Hãy thử các prompt sau:

1. `Create 3 tasks: "Setup project", "Write code", "Write tests". Make them depend on each other in order.`
2. `List all tasks and show the dependency graph`
3. `Complete task 1 and then list tasks to see task 2 unblocked`
4. `Create a task board for refactoring: parse -> transform -> emit -> test, where transform and emit can run in parallel after parse`
