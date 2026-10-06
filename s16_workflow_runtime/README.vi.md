# s16: Môi Trường Thực Thi Quy Trình — Mô Hình Quyết Định Từng Bước; Kịch Bản Quyết Định Việc Điều Phối

[English](README.md) · [中文](README.zh.md) · [日本語](README.ja.md) · [Tiếng Việt](README.vi.md)

s01 → ... → s14 → [s15](../s15_integrated_harness/) → `s16` → [s17](../s17_goal_loop/)

> *"Một tool_use vận hành toàn bộ quy trình điều phối"* — Công cụ `Workflow` khởi động một môi trường kịch bản có khả năng khôi phục, phối hợp nhiều lệnh gọi agent.
>
> **Lớp Harness**: Điều phối (Orchestration) — chạy các kịch bản đa agent đã lưu phía trên vòng lặp agent đơn lẻ.

---

Từ s01 đến s15, mô hình tự quyết định gọi công cụ nào trong mỗi vòng. Kết quả của chúng được đưa vào `messages[]`, và mô hình quyết định bước tiếp theo dựa trên ngữ cảnh đã cập nhật. Cách này hoạt động rất hiệu quả khi hướng đi tiếp theo phụ thuộc vào những gì bước trước khám phá ra.

Tuy nhiên, một số tác vụ lại lặp lại theo một trình tự cố định. Một buổi review code có thể kiểm tra đồng thời nhiều khía cạnh khác nhau, xác minh từng phát hiện, gộp các mục trùng lặp, và sắp xếp kết quả. Trình tự và các mối phụ thuộc này đã được biết trước khi thực thi. Trong trường hợp này, host cần ba yếu tố:

- **Tính song song (Parallelism)**, thay vì phải chờ đợi từng mục một cách tuần tự;
- **Cấu trúc kết quả ổn định**, ngay cả khi câu trả lời của từng agent riêng lẻ có sự khác biệt;
- **Khả năng khôi phục (Recoverability)**, để sự cố gián đoạn không làm chạy lại phần việc đã hoàn thành.

Nếu quy trình điều phối này chỉ tồn tại trong lịch sử hội thoại, thì thứ tự và các điểm kiểm tra của nó cũng chỉ tồn tại trong lịch sử đó. Một workflow đã lưu sẽ đưa trình tự cố định này vào mã nguồn và ghi nhận các lệnh gọi đã hoàn thành vào một nhật ký (journal).

## Đưa Kế Hoạch Vào Code, Không Phải Chuỗi Các Lượt Chat

Bổ sung công cụ `Workflow` vào kho công cụ của harness. Host đăng ký các kịch bản tin cậy được xây dựng từ `agent()`, `parallel()`, `pipeline()`, và `phase()`. Mô hình chỉ cần cung cấp tên workflow đã lưu, các tham số, và một mã run ID tùy chọn để tiếp tục chạy; nó không gửi mã code thực thi hay metadata.

Workflow tham gia vào vòng lặp chính dưới dạng một lệnh `tool_use`. Khi kịch bản chạy, môi trường thực thi sẽ phát ra các sự kiện vòng đời và tiến độ, đồng thời ghi lại từng bước vào một file nhật ký trên đĩa. Khi kịch bản hoàn tất, lệnh gọi sẽ trả về thông tin khởi chạy, kết quả, và trạng thái tác vụ. Kết quả trung gian của kịch bản nằm trong các biến thay vì chiếm dụng không gian trong lịch sử hội thoại. Khi được khởi động lại với `resume_from_run_id`, các lệnh gọi `agent()` không đổi sẽ truy xuất từ bộ nhớ đệm nhật ký (journal cache) và tái sử dụng kết quả trước đó.

![Workflow Runtime Overview](images/workflow-runtime-overview.svg)

```python
SAMPLE_META = {"name": "review-changes", "description": "Review code changes", "phases": ["Review", "Verify"]}

async def sample_workflow(ctx, args):
    ctx.phase("Review")
    results = await ctx.pipeline(DIMENSIONS, audit, verify)   # Mỗi chiều độc lập chạy audit → verify
    confirmed = [f for r in results if r for f in r["confirmed"]]
    ctx.log(f"Confirmed {len(confirmed)} real issues")
    return {"confirmed": confirmed}
```

## Công Cụ Workflow: Một Lệnh Gọi, Một Lượt Chạy Hoàn Chỉnh

`Workflow` được bổ sung vào kho công cụ hiện có của host s15. Người dùng có thể yêu cầu một workflow đã lưu, hoặc mô hình có thể tự chọn khi một tác vụ khớp với một quy trình điều phối đã biết. Bộ chuyển đổi (adapter) phân giải tên thông qua bảng đăng ký `WORKFLOWS` do host sở hữu, sau đó truyền metadata và hàm kịch bản tin cậy của nó vào runtime. Các công cụ s15 khác vẫn khả dụng trong cùng một vòng lặp.

Schema hiển thị cho mô hình tiếp nhận `name`, `args`, và `resume_from_run_id`. Tên không xác định hoặc tham số sai định dạng sẽ trở thành kết quả công cụ báo lỗi thay vì làm dừng vòng lặp của host. Sau đó, runtime sẽ xác thực metadata đã đăng ký, kiểm tra phân quyền, đăng ký một tác vụ workflow cục bộ, và phát ra sự kiện `async_launched` trước khi chạy kịch bản. Tiếp theo là các sự kiện tiến độ, và cuối cùng là `task_notification`; lệnh gọi trả về thông tin khởi chạy an toàn kiểu JSON, kết quả, và trạng thái tác vụ.

```python
WORKFLOW_TOOL = {
    "name": "Workflow",
    "input_schema": {
        "type": "object",
        "properties": {
            "name": {"type": "string"},
            "args": {"type": "object"},
            "resume_from_run_id": {"type": "string"},
        },
        "required": ["name"],
        "additionalProperties": False,
    },
}

async def run_workflow(name, args=None, resume_from_run_id=None):
    meta, script_fn = WORKFLOWS[name]
    out = await WorkflowTool().call(
        meta, script_fn,
        args=args,
        resume_from_run_id=resume_from_run_id,
    )
    return {"launched": out["launched"], "result": out["result"],
            "task": serialize_task(out["task"])}
```

## Metadata Của Workflow: Xác Thực Trước Khi Khởi Chạy

Mỗi workflow đã lưu đăng ký metadata tin cậy với `name`, `description`, và `phases` tùy chọn. Môi trường chạy xác thực thông tin này trước khi thực thi mã nguồn của workflow. `name` và `description` định danh tác vụ trên giao diện người dùng, trong khi `phases` đặt tên cho các nhóm trong hiển thị tiến độ. Các trường này thuộc về bảng đăng ký của host, không thuộc về đầu vào của mô hình.

Việc đăng ký không hợp lệ sẽ làm phát sinh ngoại lệ `WorkflowInputError` trước khi khởi chạy. Điều này tương tự như việc xác thực biểu thức cron ở s12: không chờ đến lúc thực thi mới phát hiện ra một workflow bị lỗi.

Vì runtime sử dụng `meta.name` trong tên file artifact cục bộ, nó cũng yêu cầu một chuỗi định danh (slug) an toàn từ 1-64 ký tự chứa chữ cái, chữ số, `.`, `_`, hoặc `-`.

```python
def validate_meta(meta):
    if not isinstance(meta, dict):
        raise WorkflowInputError("meta must be an object literal")
    if not meta.get("name") or not meta.get("description"):
        raise WorkflowInputError("meta requires name and description")
    if not isinstance(meta["name"], str) or not WORKFLOW_NAME_RE.fullmatch(meta["name"]):
        raise WorkflowInputError("meta.name must be a safe 1-64 character slug")
    if "phases" in meta and (
        not isinstance(meta["phases"], list)
        or not all(isinstance(p, str) and p for p in meta["phases"])
    ):
        raise WorkflowInputError("meta.phases must contain non-empty strings")
    return meta
```

## Các Khối Nguyên Ngữ Điều Phối (Orchestration Primitives)

Một kịch bản nhận một `ExecutionState` cung cấp một tập hợp nhỏ các khối nguyên ngữ điều phối. Nó không đọc file hay chạy lệnh shell trực tiếp. Chế độ tương tác mặc định kết nối `agent()` tới cùng một API client thực tế như host, và mỗi agent trong workflow chỉ đọc nội dung được cung cấp qua các tham số của workflow. Chế độ `demo` và các unit test sử dụng `MockAgentRunner` để các sự kiện và việc phát lại nhật ký có thể tái hiện chính xác.

| Khối nguyên ngữ | Mục đích |
|---|---|
| `agent(prompt, {schema, label, phase})` | Điều phối một subagent |
| `parallel(thunks)` | **Rào chắn (Barrier)**: chạy đồng thời tất cả các tác vụ và chờ cho đến khi toàn bộ kết quả trả về |
| `pipeline(items, *stages)` | Đưa từng mục qua các giai đoạn **mà không cần rào chắn**; mục nào xong giai đoạn trước sẽ tiếp tục ngay lập tức |
| `phase(title)` | Đánh dấu giai đoạn tiến độ hiện tại và cập nhật hiển thị tiến độ |
| `log(message)` | Phát ra một dòng log tiến độ |
| `workflow(name, args)` | Chạy một workflow con lồng nhau, chỉ hỗ trợ đúng một cấp |

Hãy dùng `pipeline` khi từng mục độc lập đi qua các giai đoạn giống nhau. Mục A có thể đến giai đoạn ba trong khi mục B vẫn ở giai đoạn một. Hãy dùng `parallel` khi bước tiếp theo cần toàn bộ kết quả từ nhóm đi trước.

```python
async def pipeline(self, items, *stages):
    async def run_item(item, idx):
        value = item
        for stage in stages:                       # Từng mục độc lập hoàn thành mỗi giai đoạn
            value = await stage(value, item, idx)
        return value
    return await asyncio.gather(*[run_item(it, i) for i, it in enumerate(items)])
```

## Đầu Ra Có Cấu Trúc: Đừng Để Subagent Trả Về Bài Văn Dài Dòng

`agent({schema})` yêu cầu một agent trong workflow chỉ trả về đối tượng JSON khớp với schema. Runtime phân tích và xác thực kết quả, sau đó thử lại một lần nếu kết quả không khớp. Mã nguồn hạ nguồn nhận được một đối tượng rõ ràng thay vì phải trích xuất các trường từ văn bản tự do.

Bài học s05 đã cảnh báo rằng các tham số công cụ không thể được tin tưởng tuyệt đối. Đây là bài học tương tự theo chiều ngược lại: đầu ra của subagent cũng không thể được tin tưởng tuyệt đối. Hãy xác thực tại ranh giới điều phối, cho phép thử lại một lần, và loại bỏ sự bất định ra khỏi phần còn lại của luồng công việc.

```python
run = await asyncio.to_thread(self.runner.run, prompt, schema, label)
result = run.value
if schema is not None:
    ok, err = SimpleJsonSchema(schema).validate(result)
    if not ok:                                       # Thử lại một lần kèm lời nhắc, sau đó mới báo lỗi
        retry = await asyncio.to_thread(
            self.runner.run, prompt + "\n\nReturn valid JSON.", schema, label
        )
        result = retry.value
        ok, err = SimpleJsonSchema(schema).validate(result)
        if not ok:
            raise WorkflowInputError(f"agent({{schema}}) returned invalid output: {err}")
```

## Trạng Thái Tác Vụ Và Sự Kiện Tiến Độ

`LocalWorkflowTask` duy trì trạng thái và mức sử dụng token, đồng thời phát ra luồng sự kiện theo chuẩn SDK: `task_started` → chuỗi các sự kiện `task_progress` chứa thông tin đổi giai đoạn, khởi động subagent và các lô log → một thông báo cuối cùng `task_notification` báo cáo hoàn thành hoặc thất bại, kèm theo file kết quả cùng số lượng agent và token đã dùng.

Chương trình demo in các sự kiện này theo thứ tự và trả về trạng thái tác vụ sau thông báo cuối cùng.

```python
class LocalWorkflowTask:
    def progress_event(self, ptype, **data):         # Phase/subagent/log
        self.progress.append({"type": ptype, **data})
        print(f"  progress   {ptype} ...")
```

## Lưu Trữ: Snapshot + Nhật Ký Để Tiếp Tục Sau Khi Bị Gián Đoạn

Runtime lưu trữ mỗi lần chạy trong thư mục `s16_workflow_runtime/.runtime/`: snapshot `<runId>.json`, kết quả đầu ra `<runId>.output.json`, nhật ký `<runId>.journal.jsonl`, và file điều phối khóa `<runId>.lock`. Mỗi lần chạy mới đặt trước một `runId` mới bằng cơ chế tạo file độc quyền trước khi mở file nhật ký của nó. Khóa chạy được giữ xuyên suốt quá trình thực thi và lưu trữ cuối cùng, do đó một tiến trình khác không thể tiếp tục cùng một lần chạy vào cùng một thời điểm. Snapshot ghi lại tên workflow, các tham số, và trạng thái tác vụ; việc tiếp tục chạy sẽ xác thực snapshot và nhật ký đã lưu trước khi thay đổi bất kỳ artifact thành công nào.

Nhật ký là cốt lõi của việc tiếp tục chạy từ điểm kiểm tra (checkpointed resume). Nó ghi lại từng kết quả `agent()` theo từng dòng:

```python
class WorkflowJournal:
    def record(self, key, value):
        self._f.write(json.dumps({"key": key, "value": value}) + "\n")
        self._f.flush()
        self.cache[key] = value
```

## Tiếp Tục Chạy (Resume): Khôi Phục Theo runId Và Tái Sử Dụng Mọi Thứ Không Đổi

Gọi lại workflow với `resume_from_run_id` sẽ chạy lại kịch bản, nhưng mỗi lệnh gọi `agent()` sẽ tính toán một khóa ngữ nghĩa xác định (deterministic semantic key). Nếu khóa đó đã có trong nhật ký, nó sẽ trả về kết quả trong cache mà không cần chạy lại mô hình. Mọi lệnh gọi không đổi đều chạm cache; chỉ có lệnh gọi bị thay đổi và các bước phụ thuộc phía sau mới thực sự chạy lại.

Điểm mấu chốt là khóa không được phụ thuộc vào thứ tự đồng thời. Các agent trong `parallel` và `pipeline` kết thúc theo thứ tự không xác định. Nếu "lượt hoàn thành thứ N" trở thành khóa, các mục cache sẽ ánh xạ sai sang các lệnh gọi khác ở lần chạy tiếp theo. Do đó, khóa sử dụng hàm băm ổn định của nội dung lệnh gọi, bao gồm loại, nhãn (label), prompt, và schema, thay vì sử dụng một bộ đếm dùng chung:

```python
def key(self, kind, label, prompt, schema):
    basis = f"{kind}|{label}|{prompt}|{json.dumps(schema, sort_keys=True)}"
    return f"{kind}-{_stable_hash(basis) % 10**10:010d}"

# Bên trong agent():
cached = self.journal.cached(key)
if cached is not MISS:
    self.task.progress_event("workflow_agent", label=label, status="cached")
    return cached
```

## Khóa Lệnh Gọi Ổn Định

Khi tiếp tục chạy, runtime phải khớp từng lệnh gọi `agent()` hiện tại với bản ghi nhật ký trước đó của nó. Một hàm băm ổn định cung cấp cùng một khóa lệnh gọi cho code workflow và các tham số không đổi. Đầu ra thực tế của mô hình có thể biến thiên; nhưng khi nội dung lệnh gọi không đổi, việc resume sẽ sử dụng luôn kết quả đã lưu trong nhật ký.

## Quan Sát Quá Trình Hoạt Động

Workflow mẫu `review-changes` sử dụng `pipeline` để gửi từng khía cạnh review một cách độc lập qua hai bước audit → verify. Chế độ tương tác sử dụng API thực tế và đọc nội dung cần review từ `args.changes`. Chế độ `demo` sử dụng dữ liệu runner cố định để minh họa đường ống, xác thực, nhật ký và hành vi resume.

```python
async def sample_workflow(ctx, args):
    ctx.phase("Review")
    changes = args.get("changes", "")

    async def audit(_v, dimension, _i):
        out = await ctx.agent(f"Inspect this change for {dimension} issues:\n{changes}",
                              schema=FINDINGS_SCHEMA, label=f"audit:{dimension}", phase="Review")
        return {"dimension": dimension, "findings": out["findings"]}

    async def verify(audited, dimension, _i):
        ctx.phase("Verify")
        verdicts = await ctx.parallel([                       # Xác minh từng phát hiện độc lập
            (lambda f=f: ctx.agent(f"Verify this finding against the change:\n{changes}\n\n{f}",
                                   schema=VERDICT_SCHEMA, label=f"verify:{dimension}:{f['title']}"))
            for f in audited["findings"]])
        return {"dimension": dimension,
                "confirmed": [f for f, v in zip(audited["findings"], verdicts) if v and v["isReal"]]}

    results = await ctx.pipeline(DIMENSIONS, audit, verify)
    ...
```

## Những Thay Đổi So Với s15

| | s15 Integrated Harness | s16 Workflow Runtime |
|--|---|---|
| Vòng lặp | Một vòng lặp do mô hình điều khiển | Vòng lặp chính không đổi; một công cụ chạy quy trình điều phối theo kịch bản |
| Ai quyết định bước tiếp theo | Mô hình quyết định ở mỗi vòng | Kịch bản khai báo trước quy trình điều phối |
| Đa agent | Các subagent s06 dùng một lần | Các lệnh gọi có thể resume, viết theo kịch bản qua ranh giới agent-runner |
| Cơ chế mới | — | Các khối nguyên ngữ kịch bản, bảng đăng ký host và adapter công cụ, vòng đời tác vụ, sự kiện tiến độ, nhật ký/resume, đầu ra có cấu trúc |

s16 không thay thế vòng lặp chính. Nó hiển thị `Workflow` ở tầng công cụ và khởi chạy một môi trường thực thi workflow cục bộ phía sau nó: một kịch bản đã lưu điều phối N lệnh gọi thông qua ranh giới agent-runner. Một subagent s06 được phân công theo quyết định tùy ý của mô hình; s16 biến quy trình điều phối thành mã nguồn host có thể khôi phục lại được.

## Thử Nghiệm

```bash
python s16_workflow_runtime/code.py          # Cả mô hình chính và các agent trong Workflow đều dùng API thật
python s16_workflow_runtime/code.py demo     # Dữ liệu mẫu review-changes xác định và luồng sự kiện
python s16_workflow_runtime/code.py resume   # Tiếp tục theo runId gần nhất; mọi agent() đều chạm cache nhật ký
```

Trong lệnh mặc định, hãy yêu cầu mô hình đọc các thay đổi, đặt nội dung văn bản đó vào `args.changes`, và chạy workflow `review-changes` đã lưu. Cả mô hình chính và các agent workflow đều sử dụng API thật. Lệnh `demo` sử dụng dữ liệu cố định để có thể quan sát vòng đời và hành vi resume một cách lặp lại. Một lần chạy demo được tiếp tục sẽ báo cáo `agents=0 tokens=0` khi mọi lệnh gọi đều chạm cache.

## Bước Tiếp Theo

[s17 Goal Loop](../s17_goal_loop/) sử dụng một vòng lặp nhỏ hơn, độc lập để kiểm tra xem mục tiêu đã nêu có đạt được hay chưa và quyết định xem có cần thêm lượt tương tác nào nữa không.

<!-- translation-sync: zh@v10, en@v10, ja@v10 -->
