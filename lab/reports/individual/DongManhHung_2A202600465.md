# Báo Cáo Cá Nhân — Lab Day 09: Multi-Agent Orchestration

**Họ và tên:** Đỗng Mạnh Hùng (MSSV: 2A202600465)  
**Vai trò trong nhóm:** MCP Owner  
**Ngày nộp:** 14/04/2026  

---

## 1. Tôi phụ trách phần nào?

Trong buổi lab này tôi tập trung vào phần MCP mock layer, cụ thể là file `mcp_server.py` và phần kết nối từ `workers/policy_tool.py` sang lớp tool này. Phần tôi theo sát nhất là cơ chế `TOOL_REGISTRY`, hàm `dispatch_tool()`, `list_tools()`, và cách đóng gói kết quả tool call thành một object có `tool`, `input`, `output`, `error`, `timestamp` để worker có thể ghi lại vào `mcp_tools_used`. Tôi cũng theo dõi contract liên quan trong `contracts/worker_contracts.yaml` để bảo đảm policy worker gọi tool theo một giao diện ổn định thay vì gọi lẫn logic domain vào trong worker.

Công việc của tôi kết nối trực tiếp với Supervisor Owner và Worker Owner. Supervisor chỉ quyết định route sang `policy_tool_worker`; sau đó worker của bạn phụ trách policy mới dùng lớp MCP mà tôi chuẩn hóa để lấy thêm dữ liệu như `search_kb`, `get_ticket_info`, hoặc `check_access_permission`. Bằng chứng rõ nhất nằm ở `mcp_server.py`, `workers/policy_tool.py`, và trong trace của file `artifacts/traces/run_20260414_155031.json`, nơi pipeline đã ghi nhận ba MCP tool calls trong cùng một câu hỏi.

## 2. Tôi đã ra một quyết định kỹ thuật gì?

Quyết định kỹ thuật quan trọng nhất của tôi là tách lớp tool ra khỏi `policy_tool_worker` bằng một cơ chế dispatch chung trong `mcp_server.py`, thay vì để worker gọi trực tiếp từng hàm xử lý riêng lẻ. Nếu đi theo cách gọi thẳng, ví dụ `policy_tool_worker` import trực tiếp `tool_search_kb()`, `tool_get_ticket_info()`, `tool_check_access_permission()`, thì code sẽ chạy được nhanh hơn ở mức ngắn hạn nhưng bị hard-code chặt vào từng tool. Khi muốn thêm tool mới, hoặc muốn thay mock bằng API thật, chúng tôi sẽ phải sửa sâu bên trong worker.

Tôi chọn cách tạo `TOOL_REGISTRY = {"search_kb": ..., "get_ticket_info": ..., ...}` và cho worker chỉ gọi qua `dispatch_tool(tool_name, tool_input)`. Trade-off của cách này là thêm một lớp trung gian, code dài hơn một chút và phải chuẩn hóa schema đầu vào/đầu ra cẩn thận. Tuy nhiên lợi ích lớn là worker chỉ biết “tôi cần tool gì”, còn MCP layer lo chuyện map tên tool sang hàm thực thi. Điều này giúp trace đẹp hơn và mở rộng dễ hơn.

Bằng chứng từ code là hàm `dispatch_tool()` trong `mcp_server.py` và `_call_mcp_tool()` trong `workers/policy_tool.py`. Bằng chứng từ trace là `run_20260414_155031.json`, trong đó `mcp_tools_used` ghi rõ ba tool đã được gọi: `search_kb`, `check_access_permission`, và `get_ticket_info`. Tôi xem đó là dấu hiệu tốt vì routing, tool call, và kết quả của từng tool đã được tách bạch đủ rõ để debug.

## 3. Tôi đã sửa một lỗi gì?

Lỗi thực tế tôi quan tâm và xử lý là trường hợp `policy_tool_worker` đi vào nhánh policy nhưng lại không có đủ context để phân tích, vì tại thời điểm đó `retrieved_chunks` có thể đang rỗng. Nếu worker cứ phân tích policy ngay trên context rỗng thì kết quả sẽ yếu, dễ rơi vào câu trả lời mơ hồ hoặc chỉ còn rule-based guessing. Vấn đề này đặc biệt rõ với các câu cần vừa policy vừa dữ liệu phụ trợ như access request đi kèm P1 incident.

Symptom là pipeline đã route đúng sang `policy_tool_worker`, nhưng nếu không có retrieval context thì `policy_result` thiếu nền tảng evidence. Root cause nằm ở chỗ worker policy không thể giả định lúc nào retrieval cũng đã chạy trước nó. Trong `graph.py`, nhánh policy hiện được gọi trực tiếp sau supervisor; vì vậy nếu muốn worker có thêm dữ liệu, chính worker phải chủ động gọi tool tìm KB.

Cách sửa là thêm bước đầu tiên trong `workers/policy_tool.py`: nếu `not chunks and needs_tool`, worker gọi `_call_mcp_tool("search_kb", {"query": task, "top_k": 3})`, sau đó lấy `output["chunks"]` gắn ngược lại vào `state["retrieved_chunks"]`. Bằng chứng sau sửa có thể thấy trong trace `run_20260414_155019.json`, nơi history ghi `[policy_tool_worker] called MCP search_kb` trước khi ghi `policy_applies=False, exceptions=1`. Điều này cho thấy worker đã biết tự kéo thêm context qua MCP thay vì chờ retrieval worker chạy riêng.

## 4. Tôi tự đánh giá đóng góp của mình

Điểm tôi làm tốt nhất là tách được vai trò giữa worker và tool layer khá rõ. Nhờ đó khi nhìn trace, tôi có thể biết worker ra quyết định gì và tool nào được dùng để bổ sung dữ liệu. Tôi cũng nghĩ mình làm tốt ở chỗ biến MCP trong lab thành một khái niệm dễ mở rộng: hôm nay là mock function, nhưng ngày mai có thể thay bằng DB, web, hay API nội bộ mà không phải viết lại toàn bộ policy worker.

Điểm tôi chưa làm tốt là phần MCP hiện vẫn mới ở mức mock in-process, chưa đẩy lên server thật hoặc HTTP transport. Ngoài ra một số semantics trong tool output vẫn còn hơi “lab style”, ví dụ `check_access_permission()` còn cần làm rõ hơn nghĩa của `can_grant=True` trong các case emergency không có bypass.

Nhóm phụ thuộc vào tôi ở phần kết nối external capability. Nếu lớp MCP chưa xong, `policy_tool_worker` chỉ còn rule-based nội bộ và sẽ rất khó xử lý các câu multi-hop. Ngược lại, phần tôi phụ thuộc vào thành viên khác là routing của supervisor và phần synthesis, vì tool có trả dữ liệu ra thì vẫn cần route đúng và tổng hợp đúng.

## 5. Nếu có thêm 2 giờ, tôi sẽ làm gì?

Nếu có thêm 2 giờ, tôi sẽ chuyển `mcp_server.py` từ mock dispatcher sang một MCP service thật hoặc ít nhất là một HTTP tool service nhỏ. Lý do là trace hiện đã cho thấy `policy_tool_worker` gọi tool tương đối ổn, nhưng toàn bộ vẫn đang chạy in-process. Nếu tách thành service thật, chúng tôi sẽ kiểm tra rõ hơn lỗi nằm ở worker logic hay ở external tool layer, nhất là với các câu multi-hop như access request đi kèm P1 incident.
