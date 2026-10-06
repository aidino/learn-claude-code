# s17: Chu Trình Hướng Mục Tiêu: Mô Hình Đề Xuất Dừng; Bộ Đánh Giá Độc Lập Quyết Định Có Tiếp Tục Hay Không

[English](README.md) · [中文](README.zh.md) · [日本語](README.ja.md) · [Tiếng Việt](README.vi.md)

s01 → ... → s15 → [s16](../s16_workflow_runtime/) → `s17`

> *"Việc mô hình không gọi thêm công cụ chỉ có nghĩa là một lượt tương tác muốn dừng lại. Một bộ đánh giá độc lập sẽ quyết định xem toàn bộ mục tiêu đã hoàn thành hay chưa."*
>
> **Lớp Harness: thực thi liên tục.** Kiểm tra điều kiện hoàn thành ở cuối mỗi lượt tương tác, và khởi động một lượt mới khi công việc vẫn còn dang dở.

---

![Goal Loop overview](images/goal-loop-overview.svg)

Kể từ s01, vòng lặp agent luôn có một điều kiện thoát đơn giản: khi mô hình dừng gọi công cụ, chương trình sẽ trả về kết quả và kết thúc.

Điều đó là đủ cho các cuộc hội thoại thông thường, nhưng không phải lúc nào cũng phù hợp cho các tác vụ như "tiếp tục sửa lỗi cho đến khi mọi bài kiểm thử đều vượt qua" hoặc "hoàn thành tất cả các tiêu chí nghiệm thu". Mô hình có thể lầm tưởng rằng công việc đã xong trong khi mới chỉ giải quyết được một phần. Việc không phát sinh `tool_use` mới chỉ có nghĩa là lượt tương tác hiện tại đã kết thúc; nó không chứng minh được rằng toàn bộ mục tiêu đã đạt được.

Lệnh `/goal` bổ sung một bước quyết định độc lập trước khi thực sự trả về kết quả.

## /goal Là Một Stop Hook Trong Phạm Vi Phiên

Nhập:

```text
/goal pytest tests/auth exits with code 0 and lint reports no errors
```

Chương trình lưu trữ điều kiện hoàn thành này và ngay lập tức gửi nó cho mô hình chính dưới dạng tác vụ hiện tại. Bạn không cần phải gửi thêm một prompt "hãy bắt đầu làm việc" thứ hai.

Khi mô hình chính dừng gọi công cụ, vòng lặp sẽ chạy hook Goal Stop trước khi trả về kết quả:

```python
if tool_results:
    messages.append({"role": "user", "content": tool_results})
    continue

decision = await self.goal.evaluate_after_turn(self.messages)
if decision.action == "block":
    self.messages.append({
        "role": "user",
        "content": decision.reason,
    })
    continue

return SessionResult(text=text, status=decision.action)
```

Khi không có mục tiêu nào đang hoạt động, hook sẽ cho phép dừng ngay lập tức, do đó điều kiện trả về hoàn toàn giống như trong s01.

## Bộ Đánh Giá Tách Biệt Với Worker

Mô hình chính chỉnh sửa code, chạy lệnh và giải quyết tác vụ. Bộ đánh giá Goal là một lệnh gọi mô hình riêng biệt với nhiệm vụ duy nhất: phán đoán điều kiện hoàn thành.

`GoalController` sở hữu bộ đánh giá như một dependency nội bộ của cổng Goal (Goal gate). Nó không phải là một đường dẫn trả về thứ hai bên cạnh vòng lặp chính.

Bài học này không có `CommandQueue` riêng: khi việc đánh giá chặn lệnh dừng, controller sẽ nối lý do vào cùng danh sách `messages[]` và bắt đầu lượt tiếp theo. Một host lớn hơn có thể sử dụng một hàng đợi chung để chuyển đầu vào của người dùng, kết quả chạy ngầm, và các lệnh tiếp tục quay lại phiên, nhưng hàng đợi đó là cơ chế truyền tải cho toàn bộ host chứ không phải thành phần thuộc quyền sở hữu của cổng Goal. Việc đặt nó bên trong cổng sẽ làm mờ ranh giới giữa quyết định và đường dẫn dùng để chuyển phát quyết định đó.

Bộ đánh giá nhìn thấy:

- Điều kiện Goal đang hoạt động;
- Toàn bộ cuộc hội thoại từ đầu đến nay;
- Kết quả công cụ mà worker đã đưa vào cuộc hội thoại đó.

Nó không có công cụ nào cả. Nó không thể tự đọc một file hay tự chạy lại một bài kiểm thử. Nó chỉ có thể phán đoán dựa trên những gì đã hiện diện trong cuộc hội thoại:

```json
{
  "ok": false,
  "reason": "The conversation does not contain pytest's exit code yet.",
  "impossible": false
}
```

`ok=true` có nghĩa là điều kiện đã được thỏa mãn. `ok=false` có nghĩa là cần thêm một lượt tương tác nữa. Nếu tác vụ không còn khả năng hoàn thành, bộ đánh giá có thể trả về `impossible=true`.

## Cuộc Hội Thoại Là Đầu Vào Của Bộ Đánh Giá

Bộ đánh giá đọc cuộc hội thoại hiện tại. Kết quả công cụ, lời giải thích của worker, và thông báo tác vụ chạy ngầm đều đi vào cuộc hội thoại dưới dạng tin nhắn, và quyết định phụ thuộc vào những gì các tin nhắn đó thực sự nói.

Đầu vào của bộ đánh giá giữ lại các tin nhắn hoàn chỉnh gần nhất. Nếu chỉ riêng tin nhắn mới nhất đã quá lớn, nó sẽ giữ lại phần đầu và phần cuối của tin nhắn đó để một kết quả công cụ duy nhất không thể chiếm hết toàn bộ yêu cầu của bộ đánh giá.

Điều đó không có nghĩa là một tuyên bố suông như "các bài kiểm thử đã qua" sẽ được chấp nhận. Prompt của bộ đánh giá yêu cầu rõ ràng các kết quả cụ thể từ cuộc hội thoại và chỉ thị mô hình không được giả định rằng một câu lệnh không báo cáo kết quả là đã thành công.

Đây vẫn là một mô hình đọc văn bản, vì vậy độ tin cậy phụ thuộc vào việc các kết quả quan trọng có được hiển thị rõ ràng hay không. System prompt của worker do đó chỉ dẫn:

> Sau khi chạy một câu lệnh xác minh, hãy báo cáo câu lệnh và kết quả của nó đủ rõ ràng để một bộ đánh giá độc lập có thể kiểm tra.

Goal Loop không phải là một framework kiểm thử. Các công cụ vẫn thực hiện việc xác minh thực tế. Bộ đánh giá Goal chỉ quyết định xem những kết quả xác minh đó đã xuất hiện trong bản ghi công việc hiện tại hay chưa.

## Một Điều Kiện Hoàn Thành Tốt Là Điều Kiện Có Thể Kiểm Tra Được

"Hãy làm cho code tốt hơn" là quá mơ hồ. Bộ đánh giá không thể biết "tốt" nghĩa là gì.

Một điều kiện hữu ích cần nêu rõ ba điều:

1. **Trạng thái kết thúc (End state):** điều gì phải đúng khi công việc hoàn thành;
2. **Kiểm tra (Check):** câu lệnh hoặc kết quả đầu ra nào chứng minh điều đó;
3. **Ràng buộc (Constraints):** điều gì không được phép làm hỏng trong quá trình thực hiện.

Ví dụ:

```text
/goal finish the authentication migration until pytest tests/auth exits 0,
without modifying test files outside tests/auth
```

Nếu bạn cần giới hạn khối lượng công việc tự động không có người giám sát, hãy sử dụng giới hạn lượt toàn cục của vòng lặp chính thay vì ẩn một ngân sách cố định bên trong Goal:

```bash
MAX_TURNS=20 python s17_goal_loop/code.py \
  "/goal fix the type errors until npm run typecheck exits 0"
```

## Công Việc Chưa Xong Quay Trở Lại Cùng Một Vòng Lặp

Khi bộ đánh giá nhận định điều kiện chưa đạt, nó sẽ trả về một lý do ngắn gọn:

```text
The conversation has no complete test result. Run pytest tests/auth and report its exit code.
```

Chương trình nối lý do đó vào `messages[]` và thực thi lệnh `continue` trong vòng lặp `while` hiện tại. Mô hình chính bắt đầu một lượt tương tác khác mà không cần đợi người dùng gõ "tiếp tục".

Không có hàng đợi tiếp tục riêng biệt nào. Việc đánh giá Goal diễn ra tại ranh giới trả về của vòng lặp, và công việc chưa hoàn thành sẽ quay trở lại thông qua chính ranh giới đó.

## Chờ Đợi Trước Khi Đánh Giá Công Việc Chạy Ngầm Chưa Hoàn Thành

Một Workflow, lệnh chạy ngầm, hoặc tác vụ bất đồng bộ khác có thể vẫn đang chạy khi mô hình chính kết thúc lượt hiện tại của nó.

Việc đánh giá ngay lúc đó sẽ là quá sớm vì kết quả quan trọng chưa quay trở lại cuộc hội thoại. Hook Goal Stop sẽ trả về `defer`, giữ cho Goal tiếp tục hoạt động, và bỏ qua bước gọi bộ đánh giá. Khi tác vụ hoàn tất, host sẽ truyền thông báo hoàn thành của nó vào `submit_background_result()`; thông báo đó đi vào cùng danh sách `messages[]`, và vòng lặp tiếp tục chạy.

Một thông báo Workflow không có đặc quyền cơ học nào. Nó đi vào cuộc hội thoại như các tin nhắn khác, và bộ đánh giá sẽ phán đoán kết quả thực tế mà nó chứa đựng.

## Tiếp Tục Tự Động Vẫn Cần Một Lối Thoát

Goal không có ngân sách mặc định ẩn gồm hai mươi lượt. Bộ đánh giá sẽ phán đoán lại điều kiện sau mỗi lượt tương tác hoàn thành.

Tuy nhiên, không có cơ chế tự động nào được phép độc chiếm một yêu cầu mãi mãi. Bài học này duy trì hai lối thoát chung bên ngoài bản thân mục tiêu:

- Giới hạn `max_turns` toàn cục của vòng lặp chính;
- Giới hạn số lần chặn liên tiếp của Stop hook.

Khi chạm giới hạn, chương trình sẽ trả lại quyền kiểm soát cho người dùng. Nó không đánh dấu mục tiêu đã hoàn thành và không âm thầm xóa mục tiêu. Người dùng có thể kiểm tra trạng thái, cung cấp thêm thông tin, cho phép tiếp tục, hoặc xóa mục tiêu.

Lỗi của bộ đánh giá cũng tuân theo quy tắc tương tự: dừng việc tự động tiếp tục, giữ nguyên mục tiêu đang hoạt động, và hiển thị lỗi thay vì tuyên bố thành công khi không thể phán đoán việc hoàn thành.

## Kiểm Tra, Thay Thế Và Xóa Bỏ

Một phiên làm việc có tối đa một Goal đang hoạt động.

```text
/goal
```

Hiển thị điều kiện, thời gian đã trôi qua, số lần đánh giá, lượng token Agent chính đã sử dụng, và lý do gần nhất của bộ đánh giá.

```text
/goal a new completion condition
```

Thay thế Goal trước đó và bắt đầu làm việc theo điều kiện mới ngay lập tức.

```text
/goal clear
```

Xóa bỏ Goal đang hoạt động. Các từ khóa `stop`, `off`, `reset`, `none`, và `cancel` cũng được chấp nhận làm bí danh tương đương.

Hàm `GoalController.restore()` có thể khôi phục một Goal vẫn đang hoạt động từ các sự kiện `goal_status` được host lưu trữ bền vững; CLI của bài học này không lưu toàn bộ phiên. Một Goal đã hoàn thành, thất bại, hoặc đã bị xóa sẽ không tự khởi động lại. Điều kiện được chuyển tiếp, trong khi số lượt, thời gian trôi qua, và mức token cơ sở sẽ bắt đầu lại từ đầu.

## Những Gì Mã Nguồn Bổ Sung

Đây là một ví dụ cơ chế độc lập được xây dựng trên nhân S04. Nó giữ nguyên năm công cụ cơ sở và bốn điểm hook, sau đó bổ sung bốn thành phần chuyên biệt cho Goal:

| Thành phần | Trách nhiệm |
|---|---|
| `GoalState` | Lưu trữ điều kiện, số lần đánh giá, thời gian bắt đầu, và lý do gần nhất |
| `PromptGoalEvaluator` | Sử dụng một lệnh gọi mô hình riêng biệt để đánh giá cuộc hội thoại |
| `GoalController` | Thiết lập, kiểm tra, xóa bỏ, và chạy hook Goal Stop |
| `AgentSession` | Kết nối hook Stop với ranh giới trả về ban đầu |

Điểm tích hợp chỉ gồm vài dòng code:

```python
decision = await self.goal.evaluate_after_turn(self.messages)
if decision.action == "block":
    continue
return SessionResult(text=text, status=decision.action)
```

## Thử Nghiệm

Cài đặt các gói phụ thuộc và chuẩn bị file `.env`:

```bash
pip install -r requirements.txt

# .env
ANTHROPIC_API_KEY=...
MODEL_ID=...

# Tùy chọn: sử dụng mô hình nhỏ hơn cho việc đánh giá Goal
GOAL_EVALUATOR_MODEL_ID=...
```

Bắt đầu phiên tương tác:

```bash
python s17_goal_loop/code.py
```

Sau đó nhập:

```text
/goal python -m pytest exits with code 0
```

Bạn cũng có thể thiết lập Goal trực tiếp từ dòng lệnh:

```bash
python s17_goal_loop/code.py "/goal python -m pytest exits with code 0"
```

## Mối Quan Hệ Với s16

s16 trả lời cho câu hỏi một lô công việc nên chạy như thế nào: các bước nào chạy đồng thời, kết quả được xác minh ra sao, và một lần chạy bị gián đoạn sẽ tiếp tục lại như thế nào.

s17 trả lời cho câu hỏi liệu toàn bộ tác vụ đã hoàn thành hay chưa. Một Workflow có thể kết thúc thành công trong khi yêu cầu cuối cùng của người dùng vẫn chưa được đáp ứng. Một khi kết quả của Workflow đi vào cuộc hội thoại, bộ đánh giá Goal sẽ quyết định xem phiên làm việc nên dừng lại hay cần tiếp tục.

Bạn có thể sử dụng riêng biệt từng cơ chế. Khi một host kết nối cả hai, thông báo hoàn thành của Workflow sẽ đi vào cuộc hội thoại và Goal Loop sẽ quyết định xem tác vụ tổng thể có cần thêm một lượt nữa hay không.

<!-- translation-sync: zh@v6, en@v6, ja@v6 -->
