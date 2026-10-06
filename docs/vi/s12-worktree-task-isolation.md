# s12: Cô Lập Tác Vụ Với Git Worktree (Worktree + Task Isolation)

`s01 > s02 > s03 > s04 > s05 > s06 | s07 > s08 > s09 > s10 > s11 > [ s12 ]`

> *"Each works in its own directory, no interference"* -- mỗi agent làm việc trong một thư mục riêng biệt của mình, không gây xung đột hay can thiệp lẫn nhau. Tác vụ quản lý mục tiêu, worktree quản lý thư mục, liên kết chặt chẽ qua ID.
>
> **Tầng Harness**: Cô lập thư mục (Directory isolation) -- các luồng thực thi song song độc lập, không bao giờ va chạm nhau.

## Vấn đề

Đến bài s11, các agent đã có thể tự chủ nhận việc và hoàn thành các tác vụ trên bảng. Tuy nhiên, mọi tác vụ đều chạy chung trong một thư mục làm việc duy nhất. Nếu hai agent cùng lúc tái cấu trúc các module khác nhau, xung đột chắc chắn xảy ra: agent A sửa `config.py`, agent B cũng sửa `config.py`, các thay đổi chưa commit bị trộn lẫn vào nhau và không ai có thể rollback sạch sẽ được.

Bảng tác vụ theo dõi *cần làm việc gì* nhưng không quản lý *làm việc đó ở đâu*. Giải pháp: cấp cho mỗi tác vụ một thư mục git worktree độc lập riêng. Tác vụ quản lý mục tiêu, worktree quản lý ngữ cảnh thực thi. Chúng được ràng buộc chặt chẽ với nhau thông qua mã định danh tác vụ (`task_id`).

## Giải pháp

```
Mặt phẳng điều khiển (.tasks/)         Mặt phẳng thực thi (.worktrees/)
+------------------+                +------------------------+
| task_1.json      |                | auth-refactor/         |
|   status: in_progress  <------>   branch: wt/auth-refactor
|   worktree: "auth-refactor"   |   task_id: 1             |
+------------------+                +------------------------+
| task_2.json      |                | ui-login/              |
|   status: pending    <------>     branch: wt/ui-login
|   worktree: "ui-login"       |   task_id: 2             |
+------------------+                +------------------------+
                                    |
                          index.json (sổ đăng ký worktree)
                          events.jsonl (nhật ký vòng đời sự kiện)

Các máy trạng thái:
  Tác vụ:     pending -> in_progress -> completed
  Worktree:   absent  -> active      -> removed | kept
```

## Cách thức hoạt động

1. **Tạo tác vụ**: Lưu trữ bền vững mục tiêu công việc trước tiên.

```python
TASKS.create("Implement auth refactor")
# -> .tasks/task_1.json  status=pending  worktree=""
```

2. **Tạo worktree và liên kết với tác vụ**: Truyền `task_id` sẽ tự động chuyển trạng thái của tác vụ sang `in_progress`.

```python
WORKTREES.create("auth-refactor", task_id=1)
# -> git worktree add -b wt/auth-refactor .worktrees/auth-refactor HEAD
# -> index.json ghi nhận mục mới, task_1.json nhận worktree="auth-refactor"
```

Quá trình liên kết ghi nhận trạng thái vào cả hai phía:

```python
def bind_worktree(self, task_id, worktree):
    task = self._load(task_id)
    task["worktree"] = worktree
    if task["status"] == "pending":
        task["status"] = "in_progress"
    self._save(task)
```

3. **Chạy các lệnh trong worktree**: Tham số `cwd` trỏ trực tiếp tới thư mục đã được cô lập.

```python
subprocess.run(command, shell=True, cwd=worktree_path,
               capture_output=True, text=True, timeout=300)
```

4. **Đóng và dọn dẹp worktree**: Có hai lựa chọn:
   - `worktree_keep(name)` -- giữ lại thư mục để kiểm tra hoặc dùng tiếp sau này.
   - `worktree_remove(name, complete_task=True)` -- gỡ bỏ thư mục, đánh dấu hoàn thành tác vụ được liên kết, và phát sự kiện. Một lệnh duy nhất xử lý trọn gói việc dọn dẹp và hoàn thành.

```python
def remove(self, name, force=False, complete_task=False):
    self._run_git(["worktree", "remove", wt["path"]])
    if complete_task and wt.get("task_id") is not None:
        self.tasks.update(wt["task_id"], status="completed")
        self.tasks.unbind_worktree(wt["task_id"])
        self.events.emit("task.completed", ...)
```

5. **Luồng sự kiện (Event stream)**: Mọi bước trong vòng đời đều ghi nhận vào `.worktrees/events.jsonl`:

```json
{
  "event": "worktree.remove.after",
  "task": {"id": 1, "status": "completed"},
  "worktree": {"name": "auth-refactor", "status": "removed"},
  "ts": 1730000000
}
```

Các sự kiện được phát ra: `worktree.create.before/after/failed`, `worktree.remove.before/after/failed`, `worktree.keep`, `task.completed`.

Kể cả khi hệ thống gặp sự cố (crash), toàn bộ trạng thái vẫn có thể tái dựng lại từ `.tasks/` + `.worktrees/index.json` trên ổ đĩa. Trí nhớ hội thoại rất dễ biến mất; chỉ có trạng thái file trên đĩa là bền vững dài lâu.

## Những điểm thay đổi so với s11

| Thành phần           | Trước đây (s11)            | Sau khi cập nhật (s12)                             |
|----------------------|----------------------------|----------------------------------------------------|
| Điều phối            | Bảng tác vụ (owner/status) | Bảng tác vụ + liên kết worktree rõ ràng            |
| Phạm vi thực thi     | Dùng chung một thư mục     | Thư mục cô lập riêng biệt theo từng tác vụ         |
| Khả năng phục hồi    | Chỉ dựa vào trạng thái task| Trạng thái task + sổ đăng ký worktree              |
| Dọn dẹp kết thúc     | Đánh dấu task completed   | Đánh dấu hoàn thành + tùy chọn giữ lại/xóa rõ ràng |
| Khả năng quan sát    | Ẩn trong log thông thường  | Sự kiện minh bạch trong `.worktrees/events.jsonl`  |

## Trải nghiệm thực tế

```sh
cd learn-claude-code
python agents/s12_worktree_task_isolation.py
```

Hãy thử các prompt sau:

1. `Create tasks for backend auth and frontend login page, then list tasks.`
2. `Create worktree "auth-refactor" for task 1, then bind task 2 to a new worktree "ui-login".`
3. `Run "git status --short" in worktree "auth-refactor".`
4. `Keep worktree "ui-login", then list worktrees and inspect events.`
5. `Remove worktree "auth-refactor" with complete_task=true, then list tasks/worktrees/events.`
