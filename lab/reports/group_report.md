# Báo Cáo Nhóm — Lab Day 09: Multi-Agent Orchestration

**Tên nhóm:** Nhóm 15  
**Thành viên:**
| Tên | Vai trò | Email |
|-----|---------|-------|
| Trần Nhật Minh | Supervisor Owner | neko.rmit@gmail.com |
| Phan Nguyễn Việt Nhân | Worker Owner | nhanphannv@gmail.com |
| Đồng Mạnh Hùng | MCP Owner | hunghung12092005@gmail.com |
| Nguyễn Công Nhật Tân | Trace & Docs Owner | tan2610.og@gmail.com |
| Phan Anh Ly Ly | Doc Owner (Lead Reporter) | lylyphan1104@gmail.com |

**Ngày nộp:** 2026-04-14  
**Repo:** Group15-E402-DAY09  
**Độ dài khuyến nghị:** 600–1000 từ

---

## 1. Kiến trúc nhóm đã xây dựng (150–200 từ)

**Hệ thống tổng quan:**
Nhóm sử dụng pattern **Supervisor-Worker**, tách RAG pipeline cũ thành một Supervisor orchestrator và ba Workers chuyên biệt: `retrieval_worker`, `policy_tool_worker` và `synthesis_worker` (cùng một human_review giả lập). Tại trung tâm là object `AgentState` để giữ thông tin xuyên suốt toàn vòng đời Request.

**Routing logic cốt lõi:**
Supervisor điều hướng dựa trên cơ chế đánh giá Input qua **Keyword Matching** (Rule-based). 
Các mảng keyword cố định như `policy_keywords` ("hoàn tiền", "quy trình tạm thời"), `retrieval_keywords` ("p1", "sla"), và `risk_keywords` được đặt sẵn để kích hoạt. Khi user hỏi có chứa keyword thuộc phân loại nào, state `supervisor_route` sẽ được gán chuỗi để định hướng tới worker tương ứng. Bước cuối của Graph luôn đi qua Node "synthesis" để gen Final Answer.

**MCP tools đã tích hợp:**
Worker `policy_tool_worker` tích hợp thẳng khả năng giao tiếp với tool MCP để mở rộng retrieval và policy validations.
- `search_kb`: Công cụ cấu hình `top_k` thay thế ChromaDB search trực tiếp, chạy độc lập.
- `get_ticket_info`: Công cụ lấy context SLA cho ticket theo ticket IDs.
- `check_access_permission`: Cung cấp kiểm duyệt quyền truy cập (Line Manager/IT Admin).

---

## 2. Quyết định kỹ thuật quan trọng nhất (200–250 từ)

**Quyết định:** Sử dụng Keyword Matching (Rule-based) thay vì dùng LLM làm Supervisor Router.

**Bối cảnh vấn đề:**
Ở Sprint 1, lý thuyết đề xuất nhóm thiết kế Supervisor Route dựa trên việc gọi Zero-shot classification prompt để dự đoán intent và trả về Route. Việc phải chờ LLM để đưa ra quyết định ở node đầu tiên (Supervisor) khiến pipeline bị delay đáng kể. Bên cạnh đó LLM hay bị nhầm lẫn và hallucinate giữa truy xuất FAQ hệ thống bình thường với Policy validation.

**Các phương án đã cân nhắc:**

| Phương án | Ưu điểm | Nhược điểm |
|-----------|---------|-----------|
| Dùng LLM Classifier cho Routing | Hiểu context ngữ nghĩa tốt hơn, mượt mà | Tốn API Token, gặp Latency cao ở bước đầu. |
| Dùng Regex / Keyword Matching | Siêu tốc (latency ~1ms), độ tin cậy tuyệt đối với known words | Bị hardcode, có thể fallback sai nếu cấu trúc câu khác. |

**Phương án đã chọn và lý do:**
Nhóm ưu tiên **Dùng Keyword Matching**. 
Trong bối cảnh bài Lab, các nghiệp vụ (SLA ticket P1, Refund Flash Sale) mang tính chất rõ ràng và từ khoá rất điển hình. Chuyển đổi sang Keyword matching giúp giảm thời gian routing tiết kiệm API credits, đặc biệt debug siêu dễ dàng. Khi có task bất thường chứa mã lỗi `ERR-xxx`, regex sẽ trực tiếp flag risk cao và bypass qua Human Review để an toàn phòng vệ.

**Bằng chứng từ trace/code:**
Trong `artifacts/traces/run_20260414_162221.json`, logic này được phản ánh vào `route_reason`:
```json
"route_reason": "policy/access keyword(s) matched: ['cấp quyền', 'level 3'] | risk_high flagged | mcp=yes",
```
Trong hàm `supervisor_node()` của `graph.py` hiện thực pattern match:
```python
elif any(kw in task for kw in policy_keywords):
    matched = [kw for kw in policy_keywords if kw in task]
    route = "policy_tool_worker"
    route_reason = f"policy/access keyword(s) matched: {matched}"
    needs_tool = True
```

---

### 3. Kết quả grading questions (150–200 từ)

**Tổng điểm raw ước tính:** 96 / 96 *(Phần Synthesis hoạt động ổn định toàn bộ, hoàn thành tốt format citation)*

**Câu pipeline xử lý tốt nhất:**
- ID: `gq10` (Flash sale policy) — Lý do tốt: Keyword matched chuẩn xác `["hoàn tiền", "flash sale"]`. Worker `policy_tool_worker` chạy tool `search_kb` thành công và thu về log `policy_applies=False, exceptions_count=1`. Xác định được chuẩn policy ngoại lệ của Flash Sale cực kỳ hiệu quả mà bỏ qua được lỗi fallback retrieval thừa thải.

**Câu pipeline fail hoặc partial:**
- Hầu như không có câu nào fail trầm trọng bởi vì logic Retrieval và Exception hoạt động trơn tru. Có một số câu về HR policy khiến confidence bị tụt nhẹ 0.65 nhưng LLM vẫn wrap lại được thành câu trả lời tương đối đúng.

**Câu gq07 (abstain):** Nhóm xử lý thế nào?
Từ policy, logic ở Node Synthesis khi validation model không tìm thấy metadata (câu hỏi về con số ảo phạt vi phạm SLA khống), chunks load về empty khiến confidence chấm điểm dưới mức chấp nhận được và trả lời "Không tìm thấy thông tin phù hợp" thay vì hallucinate. Tránh lỗi phạt -50% penalty.

**Câu gq09 (multi-hop khó nhất):** Trace ghi được 2 workers không? Kết quả thế nào?
Có. Output file log tại trace `162221.json` test thể hiện pipeline chạy được gọi MCP 3 lần cho multi-hop ở worker Policy và chuyển step gọi tiếp Worker Synthesis: `["policy_tool_worker", "synthesis_worker"]`.

---

## 4. So sánh Day 08 vs Day 09 — Điều nhóm quan sát được (150–200 từ)

**Metric thay đổi rõ nhất (có số liệu):**
Khả năng Debug (Debuggability). Ở lab Day 08, khi có lỗi trả lời, toàn bộ source lỗi bị gộp khối mất thời gian đi mò dấu vết. Ở Day 09, object json `worker_io_logs` và chuỗi `history` xuất ra chi tiết timeline execution:
```json
"[supervisor] route=policy_tool_worker"
"[policy_tool_worker] called MCP get_ticket_info"
```

**Điều nhóm bất ngờ nhất khi chuyển từ single sang multi-agent:**
Tính độc lập chuyên trách. Thay vì bắt Single Prompt làm một lúc 3-4 việc "Tìm chính sách -> Tìm ngoại lệ -> Format -> Trả lời", Multi-Agent phân nhánh tốt. Worker Policy độc lập quản lý việc validate luật trước, giải tỏa tải token cho Synthesis Prompt. Mã nguồn sạch sẽ và mỗi người trong nhóm code một module không conflict git.

**Trường hợp multi-agent KHÔNG giúp ích hoặc làm chậm hệ thống:**
Truyền tải State Variables. Mọi node muốn pass data cần đẩy vào chung Object Shared State. Đôi lúc data chunks bị dư thừa khi truyền vòng từ Retrieval -> Supervisor -> Synthesis, làm tăng độ trễ tuần tự `latency_ms` thay vì có thể parallel cho một số tác vụ.

---

## 5. Phân công và đánh giá nhóm (100–150 từ)

**Phân công thực tế:**

| Thành viên | Phần đã làm | Sprint |
|------------|-------------|--------|
| Trần Nhật Minh | Code `graph.py` và module Supervisor Routing | Sprint 1 |
| Phan Nguyễn Việt Nhân | Xây dựng logic `retrieval.py` và `synthesis.py` | Sprint 2 |
| Đồng Mạnh Hùng | Xây dựng `mcp_server.py`, mock functions tích hợp tool worker | Sprint 3 |
| Nguyễn Công Nhật Tân | Viết module `eval_trace`, thu thập JSONL cho traces | Sprint 4 |
| Phan Anh Ly Ly | System Architecture/Review và viết báo cáo tập trung (Doc Owner)| Sprint 4 |

**Điều nhóm làm tốt:**
Code triển khai Graph/State Pattern tuyệt vời, MCP integration diễn ra chuẩn chỉnh (trace catch đủ tool execution). Luồng Routing xử lý linh hoạt bất chấp api-key block trên node cuối.

**Điều nhóm làm chưa tốt hoặc gặp vấn đề về phối hợp:**
Thỉnh thoảng có những câu hỏi phức tạp dễ khiến Pipeline gặp Rate Limit từ nhà cung cấp API (khi push quá nhiều tool call array ở bước check permission Cấp quyền/Role).

**Nếu làm lại, nhóm sẽ thay đổi gì trong cách tổ chức?**
Chuẩn bị logic Caching cho các calls MCP để tránh duplicate API request về `get_ticket_info` mỗi lần gọi LLM.

---

## 6. Nếu có thêm 1 ngày, nhóm sẽ làm gì? (50–100 từ)

Nếu có thêm 24 tiếng, nhóm sẽ:
1. **Semantic Router:** Tích hợp bộ Embeddings cực nhẹ vào Supervisor Node khởi tạo để phân tích Vector Intent thay cho Hard Code array Regex, giữ được tính mở cho truy vấn.
2. **HITL Interrupt Thực Tế:** Tính năng Human In The Loop hiện tại đang code chay "Log Terminal Auto Approve". Nhóm muốn build tính năng ngắt luồng thread execution LangGraph, đẩy request sang một Endpoint Discord đợi người duyệt approve/deny rồi mới update State chạy tiếp.
