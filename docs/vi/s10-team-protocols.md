# s10: Giao Thức Phối Hợp Nhóm (Team Protocols)

`s01 > s02 > s03 > s04 > s05 > s06 | s07 > s08 > s09 > [ s10 ] > s11 > s12`

> *"Teammates need shared communication rules"* -- các thành viên trong nhóm cần có quy tắc giao tiếp chung. Một mô hình yêu cầu-phản hồi (request-response) duy nhất điều phối toàn bộ quá trình đàm phán.
>
> **Tầng Harness**: Giao thức (Protocols) -- các bắt tay có cấu trúc (structured handshakes) giữa nhiều mô hình.

## Vấn đề

Trong bài s09, các teammate có thể làm việc và giao tiếp nhưng thiếu sự phối hợp có cấu trúc:

**Tắt tiến trình an toàn (Shutdown)**: Đơn phương hủy một luồng (thread) sẽ để lại các file đang ghi dở và khiến file `config.json` trở nên lỗi thời. Bạn cần một cơ chế bắt tay: lead gửi yêu cầu, teammate chấp thuận (hoàn thành việc dở rồi thoát) hoặc từ chối (cần tiếp tục làm việc).

**Phê duyệt kế hoạch (Plan approval)**: Khi lead giao việc "hãy tái cấu trúc module xác thực auth," teammate lập tức bắt tay vào code ngay. Đối với các thay đổi có mức độ rủi ro cao, lead cần phải xem xét và duyệt qua kế hoạch trước.

Cả hai quy trình này đều chia sẻ chung một cấu trúc: một bên gửi yêu cầu kèm theo một ID định danh duy nhất (`request_id`), bên còn lại phản hồi có tham chiếu đúng tới ID đó.

## Giải pháp

```
Giao thức Shutdown               Giao thức Phê duyệt Kế hoạch
==================               ============================

Lead             Teammate        Teammate           Lead
  |                 |               |                 |
  |--shutdown_req-->|               |--plan_req------>|
  | {req_id:"abc"}  |               | {req_id:"xyz"}  |
  |                 |               |                 |
  |<--shutdown_resp-|               |<--plan_resp-----|
  | {req_id:"abc",  |               | {req_id:"xyz",  |
  |  approve:true}  |               |  approve:true}  |

Máy trạng thái hữu hạn (FSM) dùng chung:
  [pending] --approve--> [approved]
  [pending] --reject---> [rejected]

Các bộ theo dõi (Trackers):
  shutdown_requests = {req_id: {target, status}}
  plan_requests     = {req_id: {from, plan, status}}
```

## Cách thức hoạt động

1. Lead khởi xướng yêu cầu tắt an toàn bằng cách tạo một `request_id` và gửi thông qua hộp thư đến (inbox).

```python
shutdown_requests = {}

def handle_shutdown_request(teammate: str) -> str:
    req_id = str(uuid.uuid4())[:8]
    shutdown_requests[req_id] = {"target": teammate, "status": "pending"}
    BUS.send("lead", teammate, "Please shut down gracefully.",
             "shutdown_request", {"request_id": req_id})
    return f"Shutdown request {req_id} sent (status: pending)"
```

2. Teammate nhận yêu cầu và phản hồi chấp thuận hoặc từ chối (approve/reject).

```python
if tool_name == "shutdown_response":
    req_id = args["request_id"]
    approve = args["approve"]
    shutdown_requests[req_id]["status"] = "approved" if approve else "rejected"
    BUS.send(sender, "lead", args.get("reason", ""),
             "shutdown_response",
             {"request_id": req_id, "approve": approve})
```

3. Quy trình phê duyệt kế hoạch tuân theo khuôn mẫu hoàn toàn tương tự. Teammate gửi kế hoạch lên (kèm theo một `request_id`), lead đánh giá phản hồi (tham chiếu đúng `request_id` đó).

```python
plan_requests = {}

def handle_plan_review(request_id, approve, feedback=""):
    req = plan_requests[request_id]
    req["status"] = "approved" if approve else "rejected"
    BUS.send("lead", req["from"], feedback,
             "plan_approval_response",
             {"request_id": request_id, "approve": approve})
```

Một FSM duy nhất, áp dụng cho hai bài toán. Máy trạng thái `pending -> approved | rejected` xử lý tốt bất kỳ giao thức yêu cầu-phản hồi nào.

## Những điểm thay đổi so với s09

| Thành phần       | Trước đây (s09)         | Sau khi cập nhật (s10)           |
|------------------|-------------------------|----------------------------------|
| Công cụ          | 9                       | 12 (+shutdown_req/resp +plan)    |
| Tắt tiến trình   | Chỉ thoát tự nhiên      | Bắt tay yêu cầu-phản hồi         |
| Cổng duyệt kế hoạch | Không có             | Gửi/duyệt kèm quyết định phê duyệt |
| Tương quan thông điệp | Không có           | `request_id` riêng cho mỗi yêu cầu |
| Máy trạng thái FSM | Không có              | pending -> approved/rejected     |

## Trải nghiệm thực tế

```sh
cd learn-claude-code
python agents/s10_team_protocols.py
```

Hãy thử các prompt sau:

1. `Spawn alice as a coder. Then request her shutdown.`
2. `List teammates to see alice's status after shutdown approval`
3. `Spawn bob with a risky refactoring task. Review and reject his plan.`
4. `Spawn charlie, have him submit a plan, then approve it.`
5. Gõ `/team` để theo dõi trạng thái các thành viên
