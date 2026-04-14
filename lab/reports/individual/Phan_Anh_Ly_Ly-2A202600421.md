# Báo Cáo Cá Nhân — Lab Day 09: Multi-Agent Orchestration

**Họ và tên:** Phan Anh Ly Ly  
**Vai trò trong nhóm:** Doc Owner (Phác thảo kiến trúc hệ thống, Phân tích Routing và Báo cáo nhóm)  
**Ngày nộp:** 2026-04-14  
**Độ dài yêu cầu:** ~750 từ

---

## 1. Tôi phụ trách phần nào? (100–150 từ)

Ở lab này, vai trò của tôi là Doc Owner. Công việc của tôi diễn ra xuyên suốt cả 4 Sprint nhưng mạnh nhất vào Sprint 4, đặc biệt là giai đoạn Review Architecture từ baseline Day 08. 

**Module/file tôi chịu trách nhiệm:**
- Thiết kế luồng Mermaid Workflow và Schema của trạng thái chia sẻ (Shared State Schema) để các bạn trong nhóm base vào khi code qua `docs/system_architecture.md`.
- File chính diện: `reports/group_report.md`, `docs/routing_decisions.md` và góp phần review `contracts/worker_contracts.yaml`.
- Theo dõi các kết quả file Log trace trong thư mục `artifacts/traces/` để lọc ra các routing flow đạt yêu cầu, kiểm tra tính đúng đắn của pattern matching supervisor.

**Cách công việc của tôi kết nối với phần của thành viên khác:**
Kế hoạch architecture của tôi tạo nền móng để bạn Trần Nhật Minh code `graph.py` tuân theo chuẩn TypedDict cho biến `AgentState`. Các bạn code Worker sau đó chỉ cần tuân thủ Schema và nạp State Output về đúng key quy định để pipeline không ném lỗi KeyError.

---

## 2. Tôi đã ra một quyết định kỹ thuật gì? (150–200 từ)

**Quyết định:** Tôi yêu cầu module Routing của Supervisor phải thiết kế luồng log chuỗi `route_reason` dưới dạng cấu trúc chuẩn kèm Pipe (`|`), bắt buộc check cả 3 flags: keyword matched, risk_high state và mcp status, sau đó mới đẩy vào State History.

**Lý do:**
Khi chia các worker riêng lẻ ra code (người làm Retrieval, người làm Policy MCP), việc trace lại log tại sao nó lại vào các nhánh Code đó rất mơ hồ nếu Supervisor chỉ dùng một string log dạng "Route qua Policy". Việc format string rõ ràng với Pipeline Flags giúp bạn Trace & Docs dễ dàng thống kê log json cho các grading questions mà không phải lục lọi console in terminal. 

**Trade-off đã chấp nhận:**
Supervisor phải tốn thêm các thao tác xử lý nối chuỗi if/else để generate format string theo rule, khiến module `graph.py` trông có vẻ rườm rà một chút so với cách in log tự do truyền thống.

**Bằng chứng từ trace/code:**

Đoạn Json in trace log `run_20260414_162221.json` sau khi Supervisor thi hành xử lý: 
```json
{
  "supervisor_route": "policy_tool_worker",
  "route_reason": "policy/access keyword(s) matched: ['cấp quyền', 'level 3'] | risk_high flagged | mcp=yes",
  "risk_high": true,
  "needs_tool": true
}
```
Nhờ quy định output trace rule ngặt nghèo này, tôi nhặt được ngay thông tin MCP kích hoạt cho bài Report Routing Decisions.

---

## 3. Tôi đã sửa một lỗi gì? (150–200 từ)

**Lỗi:** Sự cố không đồng bộ cấu trúc đầu ra (I/O) giữa Policy_Tool_Worker và Syntax Synthesis Model.

**Symptom (pipeline làm gì sai?):**
Ở cuối phiên test đầu tiên, pipeline đi qua 2 nodes xuất ra thông tin rỗng hoặc Null ở khối Synthesis Mặc dù tool MCP `search_kb` vẫn nhặt ra được mảng chunks thành công.

**Root cause (lỗi nằm ở đâu — indexing, routing, contract, worker logic?):**
Lỗi nằm ở xung đột khai báo Model Schema trong contract. Bạn Nhân (Worker Owner) cài đặt `synthesis.py` bắt chước lấy metadata field context từ chunk lưu theo format `retrieved_chunks` (list object). Thế nhưng `policy_tool_worker` chạy tool MCP search_kb lại không mutate State trả vào mảng đó, mà trả vào một field nhỏ nằm sâu bên trong array nội bộ của MCP call history. Mâu thuẫn logic "Shared State" dẫn đến LLM bị mất dữ liệu đầu vào ngữ cảnh.

**Cách sửa:**
Với tư cách Architect, tôi đã cập nhật lại `system_architecture.md` và họp với Minh và Nhân để mapping lại logic ở Node Cuối. Các `retrieved_chunks` phải được thiết đặt gom mọi log từ `mcp_tools_used` về chung một cấu trúc Array thống nhất đặt tại root payload (Tức là `state["retrieved_chunks"]` là duy nhất và phải update liên tục) trước khi đẩy vào input Synthesis.

---

## 4. Tôi tự đánh giá đóng góp của mình (100–150 từ)

**Tôi làm tốt nhất ở điểm nào?**
Tôi vẽ sơ đồ và lên timeline cực kì chặt chẽ. Hệ thống Multi-Agent rất dễ out-sync nếu Code không đi theo thiết kế. Các doc templates đi đầu giúp ghép nối 3 module của 3 bạn Code thuận tiện, nhất là Schema cho `AgentState` TypedDict.

**Tôi làm chưa tốt hoặc còn yếu ở điểm nào?**
Do đặc thù vị trí tập trung nhiều vào logic liên kết tài liệu và đọc kết quả, tôi thiếu kỹ năng debug sâu một vài edge-case nằm ở logic prompt của Node Synthesis (đặc biệt khi Abstain bị quá nhạy đối với tài liệu HR leave_policy). Tôi cần phối hợp với Worker Owner mật thiết hơn.

**Nhóm phụ thuộc vào tôi ở đâu?**
Các báo cáo đánh giá cuối ngày của nhóm, `routing_decisions.md` để lấy rubric điểm phần Report sẽ không thể làm xong nếu tôi không gỡ rắc rối định dạng Pipeline Markdown cho nó.

**Phần tôi phụ thuộc vào thành viên khác:**
Tôi không thể phân tích report cũng như trace log được nếu bạn Trace Owner không chạy automation pass các grading questions. Tình thế Wait-For-All này khiến tôi rảnh tay vào đầu ngày và cực kì mệt vào cuối giờ nộp lab.

---

## 5. Nếu có thêm 2 giờ, tôi sẽ làm gì? (50–100 từ)

Thay vì viết báo cáo System Architecture tĩnh và vẽ sơ đồ `mermaid` thủ công, tôi sẽ tích hợp module **LangGraph Studio** hoặc tính năng **Graph.get_graph().draw_ascii()** ngay vào `graph.py` để export file Sơ đồ ra format ảnh PNG một cách tự động mỗi khi chạy test, giảm tải công việc cập nhật ảnh trong Docs. Điều này cũng giúp trace cấu trúc trực quan hơn khi hệ thống phát sinh thêm "tool worker" thứ 4, thứ 5 trong thời gian tới.
