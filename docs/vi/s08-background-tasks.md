# s08: Tác Vụ Chạy Ngầm (Background Tasks)

`s01 > s02 > s03 > s04 > s05 > s06 | s07 > [ s08 ] > s09 > s10 > s11 > s12`

> *"Run slow operations in the background; the agent keeps thinking"* -- chạy các tác vụ tốn thời gian ở chế độ ngầm; agent vẫn tiếp tục tư duy và tương tác. Các tiến trình daemon thực thi lệnh và tự động đưa thông báo kết quả vào khi hoàn tất.
>
> **Tầng Harness**: Thực thi nền (Background execution) -- mô hình tiếp tục suy luận trong khi harness chờ đợi I/O hoàn thành.

## Vấn đề

Nhiều câu lệnh tiêu tốn hàng phút để thực thi: `npm install`, `pytest`, `docker build`. Với một vòng lặp chặn (blocking loop), mô hình sẽ phải ngồi yên chờ đợi một cách lãng phí. Nếu người dùng yêu cầu "hãy cài đặt các thư viện phụ thuộc và trong lúc chờ lệnh đó chạy, hãy tạo file cấu hình config," agent dạng chặn sẽ phải làm tuần tự từng việc một thay vì thực thi song song.

## Giải pháp

```
Luồng chính (Main thread)       Luồng nền (Background thread)
+-----------------+             +-----------------+
| agent loop      |             | subprocess runs |
| ...             |             | ...             |
| [LLM call] <----+------------ | enqueue(result) |
|  ^rút queue     |             +-----------------+
+-----------------+

Dòng thời gian (Timeline):
Agent --[tạo tác vụ A]--[tạo tác vụ B]--[làm việc khác]----
             |                 |
             v                 v
        [A đang chạy]     [B đang chạy]      (song song)
             |                 |
             +-- kết quả được đưa vào trước lần gọi LLM tiếp theo --+
```

## Cách thức hoạt động

1. `BackgroundManager` quản lý các tác vụ ngầm với một hàng đợi thông báo an toàn đa luồng (thread-safe).

```python
class BackgroundManager:
    def __init__(self):
        self.tasks = {}
        self._notification_queue = []
        self._lock = threading.Lock()
```

2. Phương thức `run()` khởi chạy một daemon thread và trả về kết quả ngay lập tức mà không chặn.

```python
def run(self, command: str) -> str:
    task_id = str(uuid.uuid4())[:8]
    self.tasks[task_id] = {"status": "running", "command": command}
    thread = threading.Thread(
        target=self._execute, args=(task_id, command), daemon=True)
    thread.start()
    return f"Background task {task_id} started"
```

3. Khi tiến trình con kết thúc, kết quả của nó được đẩy vào hàng đợi thông báo.

```python
def _execute(self, task_id, command):
    try:
        r = subprocess.run(command, shell=True, cwd=WORKDIR,
            capture_output=True, text=True, timeout=300)
        output = (r.stdout + r.stderr).strip()[:50000]
    except subprocess.TimeoutExpired:
        output = "Error: Timeout (300s)"
    with self._lock:
        self._notification_queue.append({
            "task_id": task_id, "result": output[:500]})
```

4. Vòng lặp của agent rút sạch (drain) toàn bộ các thông báo trước mỗi lần gọi mô hình LLM.

```python
def agent_loop(messages: list):
    while True:
        notifs = BG.drain_notifications()
        if notifs:
            notif_text = "\n".join(
                f"[bg:{n['task_id']}] {n['result']}" for n in notifs)
            messages.append({"role": "user",
                "content": f"<background-results>\n{notif_text}\n"
                           f"</background-results>"})
        response = client.messages.create(...)
```

Vòng lặp agent vẫn duy trì đơn luồng (single-threaded). Chỉ có các tác vụ I/O của tiến trình con là được chạy song song ở chế độ nền.

## Những điểm thay đổi so với s07

| Thành phần       | Trước đây (s07)   | Sau khi cập nhật (s08)           |
|------------------|-------------------|----------------------------------|
| Công cụ          | 8                 | 6 (cơ sở + background_run + check)|
| Thực thi lệnh    | Chỉ có chặn       | Chặn + luồng chạy ngầm           |
| Thông báo        | Không có          | Hàng đợi rút thông báo mỗi vòng  |
| Đồng thời        | Không có          | Daemon threads                   |

## Trải nghiệm thực tế

```sh
cd learn-claude-code
python agents/s08_background_tasks.py
```

Hãy thử các prompt sau:

1. `Run "sleep 5 && echo done" in the background, then create a file while it runs`
2. `Start 3 background tasks: "sleep 2", "sleep 4", "sleep 6". Check their status.`
3. `Run pytest in the background and keep working on other things`
