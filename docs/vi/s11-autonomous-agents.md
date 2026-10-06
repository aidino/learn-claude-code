# s11: Agent Tự Chủ (Autonomous Agents)

`s01 > s02 > s03 > s04 > s05 > s06 | s07 > s08 > s09 > s10 > [ s11 ] > s12`

> *"Teammates scan the board and claim tasks themselves"* -- các thành viên trong đội tự quét bảng tác vụ và tự nhận việc, không cần lead phải cầm tay chỉ việc cho từng người.
>
> **Tầng Harness**: Tính tự chủ (Autonomy) -- các mô hình tự tìm việc làm mà không cần đợi chỉ định.

## Vấn đề

Trong s09-s10, các teammate chỉ làm việc khi được chỉ đạo rõ ràng. Lead phải khởi tạo từng agent kèm theo prompt cụ thể. Nếu có 10 tác vụ đang chờ trên bảng, lead phải phân công thủ công từng tác vụ một. Cách làm này không thể mở rộng quy mô (scale) được.

Tính tự chủ thực thụ đòi hỏi: các teammate tự quét bảng tác vụ, chủ động nhận các tác vụ chưa có người làm (unclaimed), thực hiện chúng, rồi tiếp tục tìm kiếm các công việc tiếp theo.

Một chi tiết tinh tế cần lưu ý: sau khi thu gọn ngữ cảnh (s06), agent có thể bị quên mất mình là ai. Kỹ thuật tái chèn định danh (identity re-injection) giải quyết triệt để vấn đề này.

## Giải pháp

```
Vòng đời của Teammate kèm chu kỳ Nghỉ (IDLE):

+-------+
| spawn |
+---+---+
    |
    v
+-------+   gọi công cụ      +-------+
| WORK  | <------------- |  LLM  |
+---+---+                +-------+
    |
    | stop_reason != tool_use (hoặc công cụ idle được gọi)
    v
+--------+
|  IDLE  |  thăm dò mỗi 5 giây trong tối đa 60 giây
+---+----+
    |
    +---> kiểm tra inbox --> có tin nhắn? ----------> WORK
    |
    +---> quét .tasks/  --> có việc chưa nhận? ----> nhận việc -> WORK
    |
    +---> quá thời gian 60 giây --------------------> SHUTDOWN

Tái chèn định danh sau khi thu gọn ngữ cảnh:
  if len(messages) <= 3:
    messages.insert(0, identity_block)
```

## Cách thức hoạt động

1. Vòng lặp của teammate gồm hai giai đoạn rõ rệt: LÀM VIỆC (WORK) và CHỜ NGHỈ (IDLE). Khi LLM dừng gọi công cụ (hoặc gọi công cụ `idle`), teammate chuyển sang trạng thái IDLE.

```python
def _loop(self, name, role, prompt):
    while True:
        # -- GIAI ĐOẠN LÀM VIỆC --
        messages = [{"role": "user", "content": prompt}]
        for _ in range(50):
            response = client.messages.create(...)
            if response.stop_reason != "tool_use":
                break
            # thực thi công cụ...
            if idle_requested:
                break

        # -- GIAI ĐOẠN CHỜ NGHỈ --
        self._set_status(name, "idle")
        resume = self._idle_poll(name, messages)
        if not resume:
            self._set_status(name, "shutdown")
            return
        self._set_status(name, "working")
```

2. Giai đoạn chờ nghỉ liên tục thăm dò (poll) hộp thư đến và bảng tác vụ theo chu kỳ.

```python
def _idle_poll(self, name, messages):
    for _ in range(IDLE_TIMEOUT // POLL_INTERVAL):  # 60s / 5s = 12 lần
        time.sleep(POLL_INTERVAL)
        inbox = BUS.read_inbox(name)
        if inbox:
            messages.append({"role": "user",
                "content": f"<inbox>{inbox}</inbox>"})
            return True
        unclaimed = scan_unclaimed_tasks()
        if unclaimed:
            claim_task(unclaimed[0]["id"], name)
            messages.append({"role": "user",
                "content": f"<auto-claimed>Task #{unclaimed[0]['id']}: "
                           f"{unclaimed[0]['subject']}</auto-claimed>"})
            return True
    return False  # hết thời gian chờ -> tắt tiến trình (shutdown)
```

3. Quét bảng tác vụ: tìm các tác vụ đang `pending`, chưa có người nhận (`owner` rỗng) và không bị chặn bởi tác vụ khác (`blockedBy` rỗng).

```python
def scan_unclaimed_tasks() -> list:
    unclaimed = []
    for f in sorted(TASKS_DIR.glob("task_*.json")):
        task = json.loads(f.read_text(encoding="utf-8"))
        if (task.get("status") == "pending"
                and not task.get("owner")
                and not task.get("blockedBy")):
            unclaimed.append(task)
    return unclaimed
```

4. Tái chèn định danh: khi ngữ cảnh quá ngắn (do vừa xảy ra thu gọn), chèn ngay một khối thông tin định danh vào đầu danh sách tin nhắn.

```python
if len(messages) <= 3:
    messages.insert(0, {"role": "user",
        "content": f"<identity>You are '{name}', role: {role}, "
                   f"team: {team_name}. Continue your work.</identity>"})
    messages.insert(1, {"role": "assistant",
        "content": f"I am {name}. Continuing."})
```

## Những điểm thay đổi so với s10

| Thành phần       | Trước đây (s10)   | Sau khi cập nhật (s11)                |
|------------------|-------------------|---------------------------------------|
| Công cụ          | 12                | 14 (+idle, +claim_task)               |
| Tính tự chủ      | Lead chỉ định     | Tự tổ chức công việc                  |
| Trạng thái nghỉ  | Không có          | Thăm dò inbox + bảng tác vụ           |
| Nhận việc        | Hoàn toàn thủ công| Tự động nhận việc chưa ai làm         |
| Định danh        | System prompt     | + Tái chèn định danh sau khi thu gọn  |
| Hết giờ (Timeout)| Không có          | 60 giây chờ -> tự động tắt (shutdown) |

## Trải nghiệm thực tế

```sh
cd learn-claude-code
python agents/s11_autonomous_agents.py
```

Hãy thử các prompt sau:

1. `Create 3 tasks on the board, then spawn alice and bob. Watch them auto-claim.`
2. `Spawn a coder teammate and let it find work from the task board itself`
3. `Create tasks with dependencies. Watch teammates respect the blocked order.`
4. Gõ `/tasks` để xem bảng tác vụ kèm danh sách người nhận việc
5. Gõ `/team` để theo dõi ai đang làm việc (working) so với ai đang nghỉ (idle)
