# s12: Bộ Lập Lịch Định Kỳ — Bắt Đầu Công Việc Theo Lịch Trình

[English](README.md) · [中文](README.zh.md) · [日本語](README.ja.md) · [Tiếng Việt](README.vi.md)

s01 → ... → s10 → s11 → `s12` → [s13](../s13_agent_teams/) → ... → s17

---

## Vấn Đề Đặt Ra

Bài s11 thay đổi cách một câu lệnh vận hành sau khi nó bắt đầu: một lệnh Bash tốn thời gian có thể chạy dưới nền. Tuy nhiên, nó không ghi nhận khi nào công việc trong tương lai nên bắt đầu, và không có thành phần nào liên tục kiểm tra thời gian hiện tại.

Đối với các yêu cầu như "chạy kiểm thử mỗi sáng lúc 9 giờ" hoặc "kiểm tra trạng thái CI mỗi 30 phút một lần", người dùng vẫn phải nhập lại prompt tại từng thời điểm đã định. Harness cần lưu trữ lịch trình, đưa prompt tương ứng vào hàng đợi chờ xử lý khi đến hạn, và chuyển phát nó tới Vòng lặp Agent khi Agent đang rảnh rỗi (idle).

---

## Giải Pháp

![Cron Scheduler Overview](images/cron-scheduler-overview.en.svg)

Giả sử Agent đăng ký một công việc định kỳ sau:

```text
cron:   0 9 * * *
prompt: run tests
```

Vào lúc 09:00 theo giờ địa phương, luồng lập lịch sẽ khớp công việc này và đẩy `[Scheduled] run tests` vào `cron_queue`. Bộ xử lý hàng đợi chờ cho đến khi Agent rảnh rỗi, sau đó bắt đầu một lượt tương tác của Vòng lặp Agent. Khi đó, mô hình có thể gọi công cụ Bash để thực thi các bài kiểm thử.

Mã nguồn S12 giữ nguyên năm công cụ cơ sở và Hooks từ S04, sau đó bổ sung `schedule_cron`, `list_crons`, và `cancel_cron`. Nó không bao gồm các lệnh chạy ngầm của S11 vì chương này chuyển phát một prompt để bắt đầu công việc, chứ không phải kết quả của một câu lệnh đang chạy.

---

## Cách Thức Hoạt Động

### Những Gì CronJob Lưu Trữ

```python
@dataclass
class CronJob:
    id: str
    cron: str
    prompt: str
    recurring: bool
    durable: bool
    pending_delivery: bool = False
    last_fired: str | None = None
```

`cron` kiểm soát thời điểm công việc đến hạn. `prompt` là tác vụ được gửi đến Agent. `pending_delivery` đánh dấu một công việc đã đến hạn nhưng mô hình chưa tiếp nhận, trong khi `last_fired` ngăn chặn việc đẩy vào hàng đợi một lần nữa trong cùng một phút.

### Biểu Thức Cron Năm Trường

```text
minute hour day month weekday
  phút  giờ ngày tháng thứ
   *     *    *    *      *    mỗi phút
   0     9    *    *      *    mỗi ngày lúc 09:00
  */5    *    *    *      *    mỗi 5 phút một lần
   0     9    *    *     1-5   các ngày trong tuần (thứ 2 - thứ 6) lúc 09:00
```

Chương này hỗ trợ `*`, `*/N`, `N`, `N-M`, và `N,M,...`. Trước khi lưu một công việc, hàm `schedule_job()` gọi `validate_cron()` và từ chối các biểu thức có số trường không đúng hoặc giá trị nằm ngoài phạm vi cho phép.

### Đẩy Vào Hàng Đợi Khi Đến Hạn

Luồng lập lịch đọc thời gian địa phương mỗi giây một lần. Khi một biểu thức khớp và công việc chưa được kích hoạt trong phút hiện tại, `_enqueue_due_job()` sẽ lưu `pending_delivery` và `last_fired` trước khi thêm công việc vào hàng đợi trong bộ nhớ:

```python
def poll_due_jobs(moment: datetime):
    minute_marker = moment.strftime("%Y-%m-%d %H:%M")
    with cron_lock:
        for job in list(scheduled_jobs.values()):
            if job.pending_delivery or job.last_fired == minute_marker:
                continue
            if cron_matches(job.cron, moment):
                _enqueue_due_job(job, minute_marker)
```

Nếu việc lưu trữ bền vững thất bại, `_enqueue_due_job()` sẽ khôi phục trạng thái trước đó và không để lộ một lượt chuyển phát chỉ tồn tại trong bộ nhớ cho bộ xử lý hàng đợi.

### Chuyển Phát Khi Agent Đang Rảnh Rỗi

`queue_processor_loop()` không kiểm tra thời gian. Nó kiểm tra hàng đợi, và khóa `agent_lock` sẽ ngăn một lượt kích hoạt theo lịch làm thay đổi phiên trong khi một lượt tương tác của người dùng đang chạy:

```python
def queue_processor_loop(stop_event=RUNTIME_STOP):
    while not stop_event.wait(0.2):
        if not has_cron_queue() or not agent_lock.acquire(blocking=False):
            continue
        try:
            if has_cron_queue():
                run_agent_turn_locked()
        finally:
            agent_lock.release()
```

Vòng lặp Agent lấy các công việc đã đến hạn ra khỏi hàng đợi và nối từng công việc vào cuộc hội thoại dưới dạng một tin nhắn mới của người dùng:

```python
fired = consume_cron_queue()
for job in fired:
    messages.append({"role": "user", "content": f"[Scheduled] {job.prompt}"})
```

Nếu lệnh gọi mô hình thất bại, những tin nhắn đó sẽ bị xóa khỏi phiên hiện tại và các công việc quay trở lại hàng đợi. Một khi mô hình đã tiếp nhận lệnh gọi thành công, các công việc chạy một lần sẽ bị xóa và các công việc định kỳ sẽ xóa cờ `pending_delivery` cho đến lần khớp tiếp theo.

### Ranh Giới Lưu Trữ Bền Vững (Persistence)

| Chế độ | Lưu trữ tại | Sau khi khởi động lại tiến trình |
|---|---|---|
| `durable=True` | `.scheduled_tasks.json` | Được nạp lại |
| `durable=False` | Bộ nhớ (RAM) | Bị mất |

Mã nguồn cập nhật `.scheduled_tasks.json` thông qua một file tạm và `os.replace()`. Nếu file bị lỗi (corrupt), quá trình khởi động sẽ báo lỗi thay vì âm thầm bỏ qua.

Cơ chế chuyển phát đảm bảo ít nhất một lần (at-least-once delivery). Nếu tiến trình thoát sau khi mô hình đã nhận prompt nhưng trước khi việc xác nhận được ghi xuống đĩa, cùng một công việc có thể sẽ được chuyển phát lại sau khi khởi động lại.

### Ranh Giới Thời Gian Chạy (Runtime Boundary)

- Bộ lập lịch sử dụng giờ địa phương của tiến trình Agent.
- Bộ lập lịch dừng lại khi tiến trình Agent thoát. Cờ `durable` chỉ bảo toàn định nghĩa công việc.
- Việc khởi động lại sẽ nạp các công việc đã lưu nhưng không chạy bù cho các mốc thời gian đã bị bỏ lỡ trong khi tiến trình tắt.
- Các lượt tương tác theo lịch chạy trong luồng của bộ xử lý hàng đợi. Một lệnh gọi công cụ cần sự phê duyệt tương tác sẽ bị từ chối thay vì cạnh tranh đầu vào với terminal chính.
- Các luồng lập lịch và xử lý hàng đợi chỉ khởi động trong giao diện dòng lệnh (CLI). Việc import file `code.py` vào module khác sẽ không khởi động luồng chạy ngầm nào.

Hãy sử dụng crontab, systemd timer, hoặc một bộ lập lịch bên ngoài nếu các công việc bắt buộc phải chạy trong lúc Agent đã đóng.

---

## Thử Nghiệm

```sh
cd learn-claude-code
python s12_cron_scheduler/code.py
```

Hãy nhập các câu lệnh prompt sau theo thứ tự:

1. `Schedule "run date" every 2 minutes and keep it after restart.`
2. `List all cron jobs.`
3. `Cancel the cron job you just created.`

Bạn có thể kiểm tra file `.scheduled_tasks.json` và theo dõi tin nhắn `[Scheduled] run date` khi công việc đến hạn. Hãy giữ tiến trình Agent tiếp tục chạy trong khi kiểm tra lịch trình ở cấp độ phút.

---

## Bước Tiếp Theo

Bộ lập lịch có thể bắt đầu một lượt tương tác của Vòng lặp Agent tại thời điểm đã định, nhưng vẫn chỉ có một Agent xử lý lượt đó. Khi một tác vụ đòi hỏi điều tra song song, sửa đổi trên nhiều module khác nhau, và tổng hợp kết quả chung, Harness cần phân công công việc cho nhiều Agent và thu thập những gì mỗi Agent tạo ra.

s13 Agent Teams → Một Agent Trưởng (Lead) phân công tác vụ, các đồng đội (teammates) chạy độc lập, và kết quả được gửi về qua hộp thư đến (inbox).

<!-- translation-sync: zh@v9, en@v9, ja@v9 -->
