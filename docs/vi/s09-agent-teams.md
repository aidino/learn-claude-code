# s09: Đội Ngũ Agent (Agent Teams)

`s01 > s02 > s03 > s04 > s05 > s06 | s07 > s08 > [ s09 ] > s10 > s11 > s12`

> *"When the task is too big for one, delegate to teammates"* -- khi tác vụ quá lớn đối với một cá nhân, hãy ủy quyền cho đồng đội. Các teammate tồn tại bền vững kết hợp với hộp thư đến (mailbox) bất đồng bộ.
>
> **Tầng Harness**: Hộp thư đội ngũ (Team mailboxes) -- nhiều mô hình phối hợp nhịp nhàng thông qua hệ thống file.

## Vấn đề

Subagent ở bài s04 chỉ mang tính dùng một lần rồi bỏ: khởi tạo, làm việc, trả về tóm tắt, rồi biến mất. Chúng không có danh tính và không lưu giữ trí nhớ giữa các lần gọi khác nhau. Tác vụ nền ở s08 thì chỉ chạy được các lệnh shell script đơn thuần chứ không thể đưa ra các quyết định dựa trên suy luận của LLM.

Làm việc nhóm thực thụ đòi hỏi: (1) các agent bền vững, sống lâu hơn một prompt đơn lẻ, (2) cơ chế quản lý vòng đời và định danh rõ ràng, (3) kênh giao tiếp trao đổi thông điệp trực tiếp giữa các agent với nhau.

## Giải pháp

```
Vòng đời của Teammate:
  spawn -> WORKING -> IDLE -> WORKING -> ... -> SHUTDOWN

Giao tiếp:
  .team/
    config.json           <- danh sách thành viên + trạng thái
    inbox/
      alice.jsonl         <- chỉ ghi nối tiếp (append-only), rút sạch khi đọc (drain-on-read)
      bob.jsonl
      lead.jsonl

              +--------+    send("alice","bob","...")    +--------+
              | alice  | -----------------------------> |  bob   |
              | loop   |    bob.jsonl << {json_line}    |  loop  |
              +--------+                                +--------+
                   ^                                         |
                   |        BUS.read_inbox("alice")          |
                   +---- alice.jsonl -> đọc + rút sạch ------+
```

## Cách thức hoạt động

1. `TeammateManager` duy trì file `config.json` chứa danh sách và trạng thái các thành viên trong đội.

```python
class TeammateManager:
    def __init__(self, team_dir: Path):
        self.dir = team_dir
        self.dir.mkdir(exist_ok=True)
        self.config_path = self.dir / "config.json"
        self.config = self._load_config()
        self.threads = {}
```

2. Phương thức `spawn()` khởi tạo một teammate mới và bắt đầu vòng lặp agent của nó trong một thread độc lập.

```python
def spawn(self, name: str, role: str, prompt: str) -> str:
    member = {"name": name, "role": role, "status": "working"}
    self.config["members"].append(member)
    self._save_config()
    thread = threading.Thread(
        target=self._teammate_loop,
        args=(name, role, prompt), daemon=True)
    thread.start()
    return f"Spawned teammate '{name}' (role: {role})"
```

3. `MessageBus`: các hộp thư JSONL ghi nối tiếp. `send()` nối thêm một dòng JSON; `read_inbox()` đọc toàn bộ tin nhắn và xóa trắng hộp thư (drain).

```python
class MessageBus:
    def send(self, sender, to, content, msg_type="message", extra=None):
        msg = {"type": msg_type, "from": sender,
               "content": content, "timestamp": time.time()}
        if extra:
            msg.update(extra)
        with open(self.dir / f"{to}.jsonl", "a") as f:
            f.write(json.dumps(msg) + "\n")

    def read_inbox(self, name):
        path = self.dir / f"{name}.jsonl"
        if not path.exists(): return "[]"
        msgs = [json.loads(l) for l in path.read_text(encoding="utf-8").strip().splitlines() if l]
        path.write_text("", encoding="utf-8")  # rút sạch
        return json.dumps(msgs, indent=2)
```

4. Mỗi teammate kiểm tra hộp thư của mình trước mỗi lần gọi LLM, đưa các thông điệp nhận được vào ngữ cảnh làm việc.

```python
def _teammate_loop(self, name, role, prompt):
    messages = [{"role": "user", "content": prompt}]
    for _ in range(50):
        inbox = BUS.read_inbox(name)
        if inbox != "[]":
            messages.append({"role": "user",
                "content": f"<inbox>{inbox}</inbox>"})
        response = client.messages.create(...)
        if response.stop_reason != "tool_use":
            break
        # thực thi công cụ, nối kết quả...
    self._find_member(name)["status"] = "idle"
```

## Những điểm thay đổi so với s08

| Thành phần       | Trước đây (s08)   | Sau khi cập nhật (s09)             |
|------------------|-------------------|------------------------------------|
| Công cụ          | 6                 | 9 (+spawn/send/read_inbox)         |
| Agent            | Đơn lẻ            | Lead + N teammates                 |
| Lưu trữ bền vững | Không có          | config.json + hộp thư JSONL        |
| Quản lý luồng    | Lệnh chạy nền     | Vòng lặp agent hoàn chỉnh mỗi thread|
| Vòng đời         | Chạy xong là kết thúc | idle -> working -> idle        |
| Giao tiếp        | Không có          | Gửi tin nhắn riêng (message) + phát sóng (broadcast) |

## Trải nghiệm thực tế

```sh
cd learn-claude-code
python agents/s09_agent_teams.py
```

Hãy thử các prompt sau:

1. `Spawn alice (coder) and bob (tester). Have alice send bob a message.`
2. `Broadcast "status update: phase 1 complete" to all teammates`
3. `Check the lead inbox for any messages`
4. Gõ `/team` để xem danh sách thành viên đội ngũ kèm trạng thái
5. Gõ `/inbox` để kiểm tra thủ công hộp thư của lead
