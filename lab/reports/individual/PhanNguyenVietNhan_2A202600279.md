# Báo Cáo Cá Nhân — Lab Day 09: Multi-Agent Orchestration

**Họ và tên:** Phan Nguyễn Việt Nhâ n
**Vai trò trong nhóm:** Worker Owner
**Ngày nộp:** 2026-04-14
**Độ dài yêu cầu:** 500–800 từ

---

## 1. Em phụ trách phần nào? (100–150 từ)

Em đảm nhận vai trò **Worker Owner** trong Sprint 2, chịu trách nhiệm trực tiếp triển khai ba worker chính của hệ thống và kết nối chúng vào orchestrator.

**Module/file em chịu trách nhiệm:**
- File chính: `workers/retrieval.py`, `workers/policy_tool.py`, `workers/synthesis.py`, `graph.py`
- Functions em implement:
  - `retrieve_dense()` — dense retrieval từ ChromaDB
  - `_llm_analyze_policy()` — LLM-based policy analysis (thêm mới hoàn toàn)
  - HITL trigger trong `synthesis.run()` khi confidence < 0.4
  - Kết nối 3 worker thật vào `graph.py` thay thế placeholder

**Cách công việc của em kết nối với phần của thành viên khác:**

Ba worker em build là tầng giữa của toàn hệ thống — supervisor (Sprint 1) route đến worker của em, và kết quả worker của em là input cho eval_trace (Sprint 4). Nếu worker trả sai format, toàn pipeline đổ.

**Bằng chứng:** Xem diff tại `graph.py` dòng 194–213 — từ placeholder hardcode sang `retrieval_run(state)`, `policy_tool_run(state)`, `synthesis_run(state)`.

---

## 2. Em đã ra một quyết định kỹ thuật gì? (150–200 từ)

**Quyết định:** Dùng **hybrid approach** cho policy analysis, rule-based làm lớp đầu tiên để detect exception nhanh, LLM làm lớp thứ hai để giải thích ngữ nghĩa sâu hơn.

Khi implement `analyze_policy()` trong `policy_tool.py`, em có ba lựa chọn:

1. **Pure rule-based** (if/else keyword matching): Nhanh, không cần API, nhưng chỉ bắt được những case đã biết trước, miss những cách diễn đạt khác nhau.
2. **Pure LLM-based**: Linh hoạt hơn, nhưng chậm, tốn token, và có thể hallucinate policy rule không có trong tài liệu.
3. **Hybrid** (em chọn): Rule-based detect exception trước → LLM giải thích kết quả dựa trên context chunk thật.

Em chọn hybrid vì nó kết hợp được độ chính xác của rule-based (exception không bị miss) với khả năng diễn giải của LLM. Đặc biệt, LLM trong `_llm_analyze_policy()` được constrain chỉ dùng context được truyền vào,không dùng kiến thức ngoài — nên tránh được hallucination.

**Trade-off đã chấp nhận:** Thêm một lần gọi API nữa per request (tăng latency ~300–500ms), nhưng chấp nhận được vì policy questions thường là high-stakes, cần giải thích rõ ràng.

**Bằng chứng từ trace/code:**

```
# policy_tool.py — kết quả test Flash Sale case
policy_applies: False
exceptions: ['flash_sale_exception']
explanation: "Policy không áp dụng cho tình huống này vì theo quy định,
              đơn hàng Flash Sale không được hoàn tiền..."

# Route log từ graph:
Route: policy_tool_worker | Reason: policy/access keyword(s) matched: ['flash sale']
Workers: ['policy_tool_worker', 'synthesis_worker']
```

---

## 3. Em đã sửa một lỗi gì? (150–200 từ)

**Lỗi:** ChromaDB index rỗng — retrieval worker trả về 0 chunks dù collection đã được tạo.

**Symptom:** Khi chạy `workers/retrieval.py`, kết quả luôn là `Retrieved: 0 chunks`, `Sources: []`. Pipeline vẫn chạy nhưng synthesis không có evidence nào, dẫn đến answer kiểu "Không đủ thông tin trong tài liệu nội bộ" cho mọi câu hỏi.

**Root cause:** Script build index trong README (Setup Step 3) bị thiếu bước quan trọng — nó chỉ đọc file và in tên (`print(f'Indexed: {fname}')`) nhưng **không bao giờ gọi `col.add()`**. Collection được tạo ra nhưng hoàn toàn rỗng. Em kiểm tra bằng `col.count()` → trả về 0.

**Cách sửa:** Em viết lại script index đúng cách:
1. Chia mỗi file thành các chunk khokho 500 ký tự với overlap 100 ký tự
2. Embed từng chunk bằng `SentenceTransformer('all-MiniLM-L6-v2')`
3. Gọi `col.add(ids, documents, embeddings, metadatas)` để thực sự lưu vào ChromaDB

**Bằng chứng trước/sau:**

```
# TRƯỚC khi sửa:
Retrieved: 0 chunks
Sources: []
⚠️ Collection 'day09_docs' chưa có data.

# SAU khi sửa:
Total chunks: 30  (5 files × ~6 chunks mỗi file)
[0.575] sla_p1_2026.txt: SLA TICKET - QUY ĐỊNH XỬ LÝ SỰ CỐ...
[0.555] sla_p1_2026.txt: Xử lý và khắc phục (resolution): 4 giờ...
Sources: ['sla_p1_2026.txt']
```

---

## 4. Em tự đánh giá đóng góp của mình (100–150 từ)

**Em làm tốt nhất ở điểm nào?**

Phần em tự tin nhất là thiết kế hybrid policy analysis — tách rõ rule-based detection (fast path) và LLM explanation (slow path), đồng thời đảm bảo LLM không hallucinate bằng cách chỉ cho phép nó dùng context được truyền vào. Ngoài ra, việc debug ChromaDB index rỗng cũng được xử lý nhanh và có bằng chứng rõ ràng.

**Em làm chưa tốt hoặc còn yếu ở điểm nào?**

Phần confidence estimation trong `synthesis.py` hiện dùng heuristic đơn giản (weighted average của chunk scores trừ exception penalty). Kết quả thực tế cho thấy confidence luôn nằm trong khoảng 0.55–0.60 cho mọi câu hỏi, không phân biệt được câu dễ và câu khó.

**Nhóm phụ thuộc vào em ở đâu?**

Synthesis worker và eval_trace phụ thuộc hoàn toàn vào output format của retrieval và policy worker. Nếu `retrieved_chunks` hay `policy_result` sai format, toàn pipeline downstream bị break.

**Phần em phụ thuộc vào thành viên khác:**

Em cần supervisor (Sprint 1) route đúng — nếu routing sai keyword, policy questions bị route sang retrieval worker và bỏ qua bước kiểm tra exception.

---

## 5. Nếu có thêm 2 giờ, em sẽ làm gì? (50–100 từ)

Em sẽ thay heuristic confidence bằng **LLM-as-Judge** thực sự. Hiện tại, trace cho thấy confidence của tất cả các câu đều xấp xỉ 0.55–0.59 — gần như không có sự phân biệt giữa câu trả lời chắc chắn và câu trả lời mơ hồ. Điều này làm mất ý nghĩa của HITL trigger (ngưỡng < 0.4 không bao giờ được kích hoạt trong điều kiện bình thường). Nếu có thêm thời gian, em sẽ implement một LLM call riêng để tự đánh giá answer có grounded với context hay không, rồi dùng kết quả đó làm confidence score thực sự.

---

*Lưu file này với tên: `reports/individual/[ten_ban].md`*
*Ví dụ: `reports/individual/nguyen_van_a.md`*
