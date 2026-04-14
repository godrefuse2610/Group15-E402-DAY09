# Báo Cáo Cá Nhân — Lab Day 09: Multi-Agent Orchestration

**Họ và tên:** Tran Nhat Minh 
**Vai trò trong nhóm:** Supervisor Owner 
**Ngày nộp:** 14/04/2026  
**Độ dài yêu cầu:** 500–800 từ

---

> **Lưu ý quan trọng:**
> - Viết ở ngôi **"tôi"**, gắn với chi tiết thật của phần bạn làm
> - Phải có **bằng chứng cụ thể**: tên file, đoạn code, kết quả trace, hoặc commit
> - Nội dung phân tích phải khác hoàn toàn với các thành viên trong nhóm
> - Deadline: Được commit **sau 18:00** (xem SCORING.md)
> - Lưu file với tên: `reports/individual/[ten_ban].md` (VD: `nguyen_van_a.md`)

---

## 1. Tôi phụ trách phần nào? (100–150 từ)

> Mô tả cụ thể module, worker, contract, hoặc phần trace bạn trực tiếp làm.
> Không chỉ nói "tôi làm Sprint X" — nói rõ file nào, function nào, quyết định nào.

**Module/file tôi chịu trách nhiệm:**
- File chính: `lab/graph.py`
- Functions tôi implement: `AgentState` (TypedDict schema), `make_initial_state`, `supervisor_node`, `route_decision`, `human_review_node`, `retrieval_worker_node`, `policy_tool_worker_node`, `synthesis_worker_node`, `build_graph`, `run_graph`, `save_trace`

**Cách công việc của tôi kết nối với phần của thành viên khác:**

`graph.py` là trung tâm điều phối — tôi định nghĩa `AgentState` mà tất cả workers đều đọc/ghi vào, đồng thời viết các wrapper node (`retrieval_worker_node`, `policy_tool_worker_node`, `synthesis_worker_node`) là điểm kết nối trực tiếp với code của các thành viên khác ở `workers/`. Khi họ implement workers thật, chỉ cần uncomment 3 dòng import và thay placeholder bằng `retrieval_run(state)`, `policy_tool_run(state)`, `synthesis_run(state)`.

**Bằng chứng (commit hash, file có comment tên bạn, v.v.):**

Commit `d3331bf` ("implement sprint 1") — toàn bộ nội dung `graph.py` từ dòng 33 đến hết được viết trong commit này. Trace `artifacts/traces/run_20260414_144721.json` và `run_20260414_144836.json` là output trực tiếp từ `run_graph()` và `save_trace()` tôi implement.

---

## 2. Tôi đã ra một quyết định kỹ thuật gì? (150–200 từ)

> Chọn **1 quyết định** bạn trực tiếp đề xuất hoặc implement trong phần mình phụ trách.
> Giải thích:
> - Quyết định là gì?
> - Các lựa chọn thay thế là gì?
> - Tại sao bạn chọn cách này?
> - Bằng chứng từ code/trace cho thấy quyết định này có effect gì?

**Quyết định:** Dùng keyword-based routing trong `supervisor_node` thay vì gọi LLM để classify task.

**Lý do:**

Có hai lựa chọn: (1) gọi LLM với prompt "classify task này thuộc nhóm nào?" — linh hoạt hơn nhưng thêm latency ~500–800ms mỗi lần và tốn token; (2) keyword matching — nhanh (<1ms), deterministic, dễ debug. Với 5 document categories trong lab và vocabulary rõ ràng (refund, P1, ERR-xxx, v.v.), keyword matching đủ chính xác. Tôi chọn cách (2) vì: trace luôn có `route_reason` cụ thể để debug (ví dụ `"policy/access keyword(s) matched: ['cấp quyền', 'level 3']"`), không phụ thuộc LLM call nên pipeline không bị chậm ở bước routing. Ngoài ra, tôi thiết kế keyword groups thành 3 danh sách riêng biệt (`policy_keywords`, `retrieval_keywords`, `risk_keywords`) để dễ mở rộng sau.

**Trade-off đã chấp nhận:**

Keyword matching không xử lý được paraphrase — ví dụ "trả lại tiền" sẽ không match "hoàn tiền" và bị route nhầm sang retrieval_worker. Đây là giới hạn chấp nhận được trong phạm vi lab với bộ test questions cố định.

**Bằng chứng từ trace/code:**

```json
// artifacts/traces/run_20260414_144721.json
"route_reason": "policy/access keyword(s) matched: ['cấp quyền', 'level 3'] | risk_high flagged",
"supervisor_route": "policy_tool_worker",
"needs_tool": true,
"risk_high": true,
"latency_ms": 0
```

```python
# graph.py:122–127
elif any(kw in task for kw in policy_keywords):
    matched = [kw for kw in policy_keywords if kw in task]
    route = "policy_tool_worker"
    route_reason = f"policy/access keyword(s) matched: {matched}"
    needs_tool = True
```

---

## 3. Tôi đã sửa một lỗi gì? (150–200 từ)

> Mô tả 1 bug thực tế bạn gặp và sửa được trong lab hôm nay.
> Phải có: mô tả lỗi, symptom, root cause, cách sửa, và bằng chứng trước/sau.

**Lỗi:** `risk_high` không được set khi task vừa match policy_keywords vừa chứa risk_keywords.

**Symptom (pipeline làm gì sai?):**

Với task `"Cần cấp quyền Level 3 để khắc phục P1 khẩn cấp. Quy trình là gì?"`, kết quả trace cho thấy `risk_high=false` dù task chứa từ "khẩn cấp" — pipeline không flag cần HITL, supervisor âm thầm route sang `policy_tool_worker` mà không cảnh báo.

**Root cause (lỗi nằm ở đâu — indexing, routing, contract, worker logic?):**

Lỗi routing logic trong `supervisor_node`. Ban đầu tôi viết Step 4 kiểm tra `risk_keywords` dưới dạng `elif`, tức là nhánh này chỉ được check khi **không** match policy_keywords hay retrieval_keywords. Kết quả: task policy + khẩn cấp → vào elif policy → skip elif risk → `risk_high` không bao giờ được set True.

**Cách sửa:**

Tách Step 4 thành `if` độc lập (không phải `elif`) để risk check luôn chạy **bất kể** route được chọn là gì. Đây là additive flag, không phải routing branch.

```python
# TRƯỚC (sai):
elif any(kw in task for kw in risk_keywords):
    risk_high = True

# SAU (đúng — graph.py:135):
if any(kw in task for kw in risk_keywords):    # ← if, không phải elif
    risk_high = True
    route_reason += " | risk_high flagged"
```

**Bằng chứng trước/sau:**

```
# TRƯỚC khi sửa (risk_high bị miss):
route_reason: "policy/access keyword(s) matched: ['cấp quyền', 'level 3']"
risk_high: false   ← SAI — "khẩn cấp" không được detect

# SAU khi sửa (trace run_20260414_144721.json):
route_reason: "policy/access keyword(s) matched: ['cấp quyền', 'level 3'] | risk_high flagged"
risk_high: true    ← ĐÚNG
```

---

## 4. Tôi tự đánh giá đóng góp của mình (100–150 từ)

> Trả lời trung thực — không phải để khen ngợi bản thân.

**Tôi làm tốt nhất ở điểm nào?**

Thiết kế `AgentState` schema và routing logic trong `supervisor_node` — state có đủ fields để trace toàn bộ pipeline (route_reason, workers_called, history, latency_ms), routing additive risk_high flag hoạt động đúng sau khi sửa bug. `save_trace` tự động persist ra JSON cho mỗi run giúp debug dễ.

**Tôi làm chưa tốt hoặc còn yếu ở điểm nào?**

Keyword list hiện còn cứng và tiếng Việt không đồng nhất — "trả lại tiền" sẽ không match "hoàn tiền". Tôi chưa xử lý được trường hợp task ambiguous (không match keyword nào nhưng cũng không rõ là retrieval). Ngoài ra chưa implement LangGraph Option B mặc dù đã có TODO.

**Nhóm phụ thuộc vào tôi ở đâu?** _(Phần nào của hệ thống bị block nếu tôi chưa xong?)_

Toàn bộ pipeline bị block nếu `graph.py` chưa có: `AgentState` định nghĩa contract cho tất cả workers, `build_graph()` là entry point để chạy end-to-end, và `save_trace()` là cơ chế lưu kết quả để eval.

**Phần tôi phụ thuộc vào thành viên khác:** _(Tôi cần gì từ ai để tiếp tục được?)_

Tôi cần Sprint 2 của các thành viên implement xong `workers/retrieval.py`, `workers/policy_tool.py`, `workers/synthesis.py` thì mới uncomment import và thay placeholder nodes bằng worker thật được. Hiện tại graph chạy được nhưng `final_answer` chỉ là placeholder.

---

## 5. Nếu có thêm 2 giờ, tôi sẽ làm gì? (50–100 từ)

> Nêu **đúng 1 cải tiến** với lý do có bằng chứng từ trace hoặc scorecard.
> Không phải "làm tốt hơn chung chung" — phải là:
> *"Tôi sẽ thử X vì trace của câu gq___ cho thấy Y."*

Tôi sẽ chuyển `build_graph()` sang dùng LangGraph `StateGraph` với conditional edges thật sự (Option B trong TODO). Lý do: trace của `run_20260414_144721` cho thấy `latency_ms=0` — pipeline hiện chạy sequential Python thuần không có async, không có parallel execution. LangGraph cho phép `retrieval_worker` và `policy_tool_worker` chạy song song thay vì tuần tự (dòng 279–284 trong `build_graph`), giảm latency thực tế khi có cả hai workers cần chạy.

---

