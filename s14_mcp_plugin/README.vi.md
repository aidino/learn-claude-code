# s14: Công Cụ MCP — Khám Phá Và Triệu Gọi Công Cụ Bên Ngoài

[English](README.md) · [中文](README.zh.md) · [日本語](README.ja.md) · [Tiếng Việt](README.vi.md)

[s04](../s04_hooks/) → `s14` → [s15](../s15_integrated_harness/) → s16 → s17

> **Lớp Harness**: Công cụ MCP (MCP Tools) — kết nối với các dịch vụ, khám phá công cụ và đưa chúng vào vòng lặp agent.

---

## Vấn Đề Đặt Ra

Các công cụ cơ sở trong các chương trước được viết trực tiếp trong file `code.py`. Chúng ta có thể tích hợp một hệ thống tài liệu và một nền tảng triển khai bằng cách thêm `search_docs`, `deploy_status`, và `trigger_deploy`, nhưng mỗi dịch vụ sẽ đòi hỏi một tập hợp các định nghĩa công cụ, schema tham số và hàm xử lý triệu gọi riêng.

Giao thức MCP (Model Context Protocol) tách biệt các trách nhiệm đó. Một server MCP cung cấp danh sách công cụ và endpoint để triệu gọi. Harness kết nối với server đó, gán tên hiển thị phù hợp cho mô hình, áp dụng các kiểm tra phân quyền, và cung cấp các công cụ vừa khám phá được cho mô hình sử dụng.

---

## Giải Pháp

![MCP Architecture](images/mcp-architecture.en.svg)

Chương này bắt đầu từ năm công cụ cơ sở và hooks của s04, sau đó bổ sung ba thành phần:

- `MCPClient` lưu trữ các định nghĩa công cụ và hàm xử lý triệu gọi do một server trả về.
- `connect_mcp` kết nối tới một server và lấy danh sách công cụ của server đó.
- `assemble_tool_pool` kết hợp các công cụ cơ sở với các công cụ từ mọi server đang kết nối.

Các server `docs` và `deploy` ở đây là các đối tượng giả lập trong tiến trình (in-process stand-ins) đại diện cho `tools/list`, `tools/call`, và một kho công cụ động. Chương này không triển khai một tầng truyền tải (transport) MCP thực tế qua mạng hay tiến trình con.

---

## Cách Thức Hoạt Động

### 1. Vòng Lặp Agent Cơ Sở Được Giữ Nguyên

Trước mỗi lệnh gọi mô hình, harness sẽ tập hợp kho công cụ hiện tại:

```python
def agent_loop(messages: list):
    while True:
        tools, handlers = assemble_tool_pool()
        response = client.messages.create(
            model=MODEL,
            system=assemble_system_prompt(),
            messages=messages,
            tools=tools,
            max_tokens=8000,
        )
        ...
```

Sau khi một server mới kết nối thành công, lệnh gọi `assemble_tool_pool()` tiếp theo sẽ thêm các công cụ của server đó vào đầu vào của mô hình. Kết quả của công cụ vẫn được nối vào danh sách `messages` dưới dạng các khối `tool_result`.

### 2. MCPClient Lưu Trữ Kết Quả Khám Phá Và Hàm Xử Lý

```python
class MCPClient:
    def register(self, tool_defs, handlers):
        self.tools = list(tool_defs)
        self._handlers = dict(handlers)

    def call_tool(self, tool_name, args):
        handler = self._handlers.get(tool_name)
        if not handler:
            return f"MCP error: unknown tool '{tool_name}'"
        try:
            return str(handler(**args))
        except Exception as error:
            return f"MCP error: {type(error).__name__}: {error}"
```

`register()` đại diện cho danh sách công cụ đã khám phá. `call_tool()` đại diện cho ranh giới triệu gọi. Các lỗi phát sinh sẽ trả về cho mô hình thay vì làm dừng đột ngột vòng lặp agent.

### 3. connect_mcp Chỉ Kết Nối Và Khám Phá

```python
def connect_mcp(name: str) -> str:
    if name in mcp_clients:
        return f"MCP server '{name}' already connected"
    factory = MOCK_SERVERS.get(name)
    if not factory:
        return f"Unknown server '{name}'"
    server = factory()
    mcp_clients[name] = server
    ...
```

Ban đầu, mô hình chỉ nhìn thấy năm công cụ cơ sở và `connect_mcp`. Sau khi gọi `connect_mcp(name="docs")`, harness lưu trữ client docs. Lệnh gọi mô hình tiếp theo sẽ thấy thêm:

```text
mcp__docs__search
mcp__docs__get_version
```

### 4. Tiền Tố (Prefix) Phân Tách Công Cụ Giữa Các Server Khác Nhau

Nhiều server khác nhau đều có thể cung cấp công cụ `search` hoặc `status`. Harness sử dụng quy ước đặt tên:

```text
mcp__{server}__{tool}
```

Hàm `normalize_mcp_name()` thay thế các ký tự nằm ngoài bảng chữ cái tên công cụ của mô hình bằng dấu gạch dưới (`_`). Quá trình tập hợp kho công cụ cũng kiểm tra sự xung đột tên sau khi chuẩn hóa và giới hạn độ dài 64 ký tự:

```python
prefixed = f"mcp__{safe_server}__{safe_tool}"
if prefixed in origins:
    raise ValueError("MCP tool name collision after normalization")
```

Nhờ đó, `docs.one/get.version` và `docs_one/get_version` không thể bị ánh xạ trùng lặp một cách âm thầm vào cùng một tên.

### 5. Định Nghĩa Công Cụ Và Hàm Xử Lý Cùng Đi Vào Kho Công Cụ

```python
tools.append({
    "name": prefixed,
    "description": tool_def.get("description", ""),
    "input_schema": schema,
})
handlers[prefixed] = (
    lambda *, client=server, tool=raw_name, **kwargs:
    client.call_tool(tool, kwargs)
)
```

Mô hình nhìn thấy tên có tiền tố. Hàm xử lý sẽ gọi `MCPClient` với tên công cụ gốc của server. Các tham số mặc định (`client=server, tool=raw_name`) bắt giữ (capture) client và công cụ hiện tại để tránh việc mọi hàm lambda đều trỏ tới phần tử cuối cùng trong vòng lặp.

### 6. Phía Host Quyết Định Quyền Thực Thi

Một server MCP có thể cung cấp `readOnlyHint` hoặc `destructiveHint`, nhưng những gợi ý đó xuất phát từ phía server và không phải là sự cấp quyền. Chương này sử dụng chính sách kiểm soát phía host:

```python
MCP_HOST_POLICY = {
    ("docs", "search"): "allow",
    ("docs", "get_version"): "allow",
    ("deploy", "status"): "allow",
    ("deploy", "trigger"): "confirm",
}
```

`permission_hook()` tra cứu chính sách này bằng tên công cụ đã được chuẩn hóa. Mặc định, một công cụ bên ngoài chưa được cấu hình sẽ yêu cầu người dùng xác nhận. Một mô tả chứa chữ `readOnly` không đồng nghĩa với việc công cụ đó được tin cậy.

### 7. Lỗi Tham Số Đầu Vào Dừng Lại Ở Ranh Giới Công Cụ

Mô hình có thể bỏ sót một đối số bắt buộc hoặc gửi một trường mà server không chấp nhận. Cả `execute_tool()` và `MCPClient.call_tool()` đều bắt các lỗi đó và trả về một `tool_result` chứa thông báo lỗi:

```text
MCP error: TypeError: <lambda>() missing 1 required argument: 'query'
```

Mô hình có thể sửa lại các đối số trong lượt tương tác tiếp theo mà không làm gián đoạn kịch bản của bài học.

---

## Những Thay Đổi So Với s04

| Thành phần | s04 | s14 |
|---|---|---|
| Công cụ cơ sở | Năm công cụ cố định | Không đổi |
| Nguồn công cụ | Các định nghĩa trong `code.py` | Công cụ cơ sở cộng với các công cụ MCP được khám phá |
| Kho công cụ | `TOOLS` cố định | Được tạo lại mỗi lượt bởi `assemble_tool_pool()` |
| Tên công cụ ngoài | Không có | `mcp__{server}__{tool}` |
| Phân quyền | Kiểm tra Shell và đường dẫn | Bổ sung chính sách MCP phía host |
| Tầng truyền tải MCP | Không có | Các đối tượng server giả lập trong tiến trình thể hiện ranh giới |

Chương này không bao gồm Tác vụ (Task), Chạy ngầm (Background), Cron, Đội ngũ (Team), hay Worktree. Chúng sẽ kết hợp cùng với MCP trong Khung điều hành tích hợp ở bài s15.

---

## Thử Nghiệm

```sh
cd learn-claude-code
python s14_mcp_plugin/code.py
```

Nhập:

```text
Connect to the docs server, search for agent hooks, and tell me the current documentation API version.
```

Dấu vết gọi công cụ thông thường sẽ là:

```text
connect_mcp(name="docs")
mcp__docs__search(query="agent hooks")
mcp__docs__get_version()
```

Sau đó nhập:

```text
Connect to the deploy server and check the web service status. Do not trigger a deployment.
```

Công cụ `status` chạy theo chính sách cho phép của host. Công cụ `trigger` sẽ yêu cầu người dùng xác nhận.

---

## Bước Tiếp Theo

Ở đây, MCP vẫn là một nhánh bài học độc lập. s15 Integrated Harness sẽ kết hợp các công cụ cơ sở, hooks, kỹ năng (skills), ngữ cảnh (context), bộ nhớ (memory), tác vụ (tasks), tác vụ chạy ngầm, cron, đội ngũ (teams) và MCP vào chung một môi trường thực thi duy nhất.

<!-- translation-sync: zh@v9, en@v9, ja@v9 -->
