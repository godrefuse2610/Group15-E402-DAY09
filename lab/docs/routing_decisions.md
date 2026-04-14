# Routing Decisions Log — Lab Day 09

**Nhóm:** Nhóm 15  
**Ngày:** 2026-04-14

> **Hướng dẫn:** Ghi lại ít nhất **3 quyết định routing** thực tế từ trace của nhóm.
> Không ghi giả định — phải từ trace thật (`artifacts/traces/`).
> 
> Mỗi entry phải có: task đầu vào → worker được chọn → route_reason → kết quả thực tế.

---

## Routing Decision #1

**Task đầu vào:**
> Sự cố P1 xử lý trong bao lâu? (Query: "SLA xử lý ticket P1 là bao lâu?")

**Worker được chọn:** `retrieval_worker`  
**Route reason (từ trace):** `retrieval keyword(s) matched: ['p1', 'sla', 'ticket'] | mcp=no`  
**MCP tools được gọi:** Không có  
**Workers called sequence:** `retrieval_worker`, `synthesis_worker`

**Kết quả thực tế:**
- final_answer (ngắn): `SLA xử lý ticket P1 là 4 giờ kể từ khi ticket được tạo. Phản hồi ban đầu là 15 phút. [sla_p1_2026.txt]`
- confidence: `0.58`
- Correct routing? Yes

**Nhận xét:** _(Routing này đúng hay sai? Nếu sai, nguyên nhân là gì?)_

Routing logic xử lý đúng nhờ keyword matching `p1`, `sla`, `ticket` nên đã chuyển tiếp trực tiếp vào `retrieval_worker`. Không cần gọi tool phức tạp, hệ thống sinh ra câu trả lời chuẩn xác. Quy trình nhanh gọn và đạt yêu cầu.

---

## Routing Decision #2

**Task đầu vào:**
> Khách hàng có thể yêu cầu hoàn tiền trong bao nhiêu ngày?

**Worker được chọn:** `policy_tool_worker`  
**Route reason (từ trace):** `policy/access keyword(s) matched: ['hoàn tiền'] | mcp=yes`  
**MCP tools được gọi:** `search_kb`  
**Workers called sequence:** `policy_tool_worker`, `synthesis_worker`

**Kết quả thực tế:**
- final_answer (ngắn): `Khách hàng có thể yêu cầu hoàn tiền trong vòng 7 ngày làm việc kể từ thời điểm xác nhận... [policy_refund_v4.txt]`
- confidence: `0.65`
- Correct routing? Yes

**Nhận xét:**

Đây là quyết định đúng dựa trên keyword `hoàn tiền` yêu cầu gọi `policy_tool_worker`. Trace thể hiện worker này cũng đã gọi thành công MCP tool `search_kb` và trả về `policy_applies=True, exceptions=0`. Hệ thống đã nhận diện được chính sách đúng đắn và LLM tổng hợp ra đáp án 7 ngày.

---

## Routing Decision #3

**Task đầu vào:**
> Cần cấp quyền Level 3 để khắc phục P1 khẩn cấp. Quy trình là gì?

**Worker được chọn:** `policy_tool_worker`  
**Route reason (từ trace):** `policy/access keyword(s) matched: ['cấp quyền', 'level 3'] | risk_high flagged | mcp=yes`  
**MCP tools được gọi:** `search_kb`, `check_access_permission`, `get_ticket_info`  
**Workers called sequence:** `policy_tool_worker`, `synthesis_worker`

**Kết quả thực tế:**
- final_answer (ngắn): `Quy trình yêu cầu phê duyệt Line Manager, IT Admin, IT Security...`
- confidence: `0.62`
- Correct routing? Yes

**Nhận xét:**

Xử lý routing rất chính xác. Task có chứa `cấp quyền`, `level 3` cũng như keyword `khẩn cấp` (`risk_high`), đòi hỏi gọi MCP. Quá trình chọn `policy_tool_worker` giúp tích hợp các tool như kiểm tra quyền (`check_access_permission`) và lấy thông tin ticket P1 (`get_ticket_info`), trả lời đúng theo workflow của Access Control SOP.

---

## Tổng kết

### Routing Distribution

| Worker | Số câu được route | % tổng |
|--------|------------------|--------|
| retrieval_worker | 1 | 33.3% |
| policy_tool_worker | 2 | 66.7% |
| human_review | 0 | 0% |

*(Thống kê dựa trên 3 runs ngẫu nhiên ở trên)*

### Routing Accuracy

> Trong số 3 câu nhóm đã chạy, bao nhiêu câu supervisor route đúng?

- Câu route đúng: 3 / 3
- Câu route sai (đã sửa bằng cách nào?): 0
- Câu trigger HITL: 0

### Lesson Learned về Routing

> Quyết định kỹ thuật quan trọng nhất nhóm đưa ra về routing logic là gì?  
> (VD: dùng keyword matching vs LLM classifier, threshold confidence cho HITL, v.v.)

1. Ưu tiên Keyword Matching thay vì LLM: Nhóm chọn rules/keywords cho routing logic để tối ưu tốc độ `latency_ms` và xử lý deterministic hơn LLM (tránh API latency ở khâu routing).
2. Xử lý MCP theo Route Reason: Khả năng MCP cần linh hoạt gọi khi keyword match với các nhóm "policy", "access", thay vì gọi vô tội vạ. 

### Route Reason Quality

> Nhìn lại các `route_reason` trong trace — chúng có đủ thông tin để debug không?  
> Nếu chưa, nhóm sẽ cải tiến format route_reason thế nào?

Route reason ghi đủ thông tin: rule matched, keywords detect được (`['cấp quyền', 'level 3']`), `risk_high` status và `mcp=yes/no`. Nhờ format này, việc debug rất rõ ràng và dễ đối chiếu với yêu cầu đầu vào. Để cải tiến, có thể bổ sung thêm confidence score của classify vào route_reason nếu có tích hợp classifier ở phase sau.
