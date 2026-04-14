# Single Agent vs Multi-Agent Comparison — Lab Day 09

**Nhóm:** _____15______  
**Ngày:** ______14/4/2026_____

> **Hướng dẫn:** So sánh Day 08 (single-agent RAG) với Day 09 (supervisor-worker).
> Phải có **số liệu thực tế** từ trace — không ghi ước đoán.
> Chạy cùng test questions cho cả hai nếu có thể.

---

## 1. Metrics Comparison

> Điền vào bảng sau. Lấy số liệu từ:
> - Day 08: chạy `python eval.py` từ Day 08 lab
> - Day 09: chạy `python eval_trace.py` từ lab này

| Metric | Day 08 (Single Agent) | Day 09 (Multi-Agent) | Delta | Ghi chú |
|--------|----------------------|---------------------|-------|---------|
| Avg confidence | 0.00 | 0.609 | +0.609 | Day 08 không đếm, Day 09 do AI tự chấm |
| Avg latency (ms) | 2469 ms | 16347 ms | +13878 ms | Day 09 chậm hơn do phải loop qua Supervisor -> Tools |
| Abstain rate (%) | N/A | 5% (HITL rate) | N/A | Day 09 có cơ chế Human-in-the-loop khi bí |
| Multi-hop accuracy | N/A | N/A | N/A | % câu multi-hop trả lời đúng |
| Routing visibility | ✗ Không có | ✓ Có route_reason | N/A | Giúp debug dễ hơn |
| Debug time (estimate) | ~15 phút | ~3 phút | Nhanh hơn | Nhờ Trace file JSON chỉ đúng lỗi ở chặng nào |
| Khả năng mở rộng | Khó | Plug-and-Play | N/A | Thêm tool vào server MCP dễ dàng |

> **Lưu ý:** Nếu không có Day 08 kết quả thực tế, ghi "N/A" và giải thích.

---

## 2. Phân tích theo loại câu hỏi

### 2.1 Câu hỏi đơn giản (single-document)

| Nhận xét | Day 08 | Day 09 |
|---------|--------|--------|
| Accuracy | Cao | Cao |
| Latency | Rất Nhanh (~2.5s) | Chậm hơn (~16s) |
| Observation | Xử lý một mạch, lấy data và trả lời ngay. | Mất thêm thời gian ghé qua Supervisor phân loại rồi mới về Retrieval. |

**Kết luận:** Multi-agent KHÔNG CẢI THIỆN với câu hỏi quá đơn giản. Thậm chí làm hệ thống chậm hơn do overhead của routing.

_________________

### 2.2 Câu hỏi multi-hop (cross-document)

| Nhận xét | Day 08 | Day 09 |
|---------|--------|--------|
| Accuracy | Trung bình | Tốt hơn |
| Routing visible? | ✗ | ✓ |
| Observation | LLM bị rối vì context nhét chung. | Phân ra worker xử lý riêng, ráp qua MCP gọn gàng. |

**Kết luận:** Multi-agent có ưu thế vượt trội khi xử lý nghiệp vụ dài (như check Policy Flash Sale).

_________________

### 2.3 Câu hỏi cần abstain

| Nhận xét | Day 08 | Day 09 |
|---------|--------|--------|
| Abstain rate | N/A | 5% |
| Hallucination cases | Dễ bịa ra câu trả lời sai | Ít bịa hơn nhờ cơ chế rẽ nhánh |
| Observation | Ép LLM phải sinh ra câu trả lời bằng mọi giá | Có Supervisor đẩy "unknown error code" vào cổng Human-In-The-Loop. |

**Kết luận:** Day 09 an toàn hơn cho triển khai thực tế.

_________________

---

## 3. Debuggability Analysis

> Khi pipeline trả lời sai, mất bao lâu để tìm ra nguyên nhân?

### Day 08 — Debug workflow
```
Khi answer sai → phải đọc toàn bộ RAG pipeline code → tìm lỗi ở indexing/retrieval/generation
Không có trace → không biết bắt đầu từ đâu
Thời gian ước tính: ~15 phút
```

### Day 09 — Debug workflow
```
Khi answer sai → đọc trace → xem supervisor_route + route_reason
  → Nếu route sai → sửa supervisor routing logic
  → Nếu retrieval sai → test retrieval_worker độc lập
  → Nếu synthesis sai → test synthesis_worker độc lập
Thời gian ước tính: ~3 phút
```

**Câu cụ thể nhóm đã debug:** _(Mô tả 1 lần debug thực tế trong lab)_
- Lỗi tải model SentenceTransformer lặp đi lặp lại nhiều lần ở mỗi Worker gây Timeout và Treo luồng (KeyboardInterrupt).
- **Cách tìm ra:** Xác định qua Trace log đang kẹt ở khâu `retrieval`, sau đó nhóm sửa lại code theo kiểu Global biến `_STATIC_HF_MODEL` để khắc phục vĩnh viễn sự cố tải.

_________________

---

## 4. Extensibility Analysis

> Dễ extend thêm capability không?

| Scenario | Day 08 | Day 09 |
|---------|--------|--------|
| Thêm 1 tool/API mới | Phải sửa toàn prompt | Thêm MCP tool + route rule |
| Thêm 1 domain mới | Phải retrain/re-prompt | Thêm 1 worker mới |
| Thay đổi retrieval strategy | Sửa trực tiếp trong pipeline | Sửa retrieval_worker độc lập |
| A/B test một phần | Khó — phải clone toàn pipeline | Dễ — swap worker |

**Nhận xét:**
Day 09 hỗ trợ dạng Plug-and-Play (gắn thêm/rút độc lập). Điều này rất hữu ích khi mở rộng Project lớn.
_________________

---

## 5. Cost & Latency Trade-off

> Multi-agent thường tốn nhiều LLM calls hơn. Nhóm đo được gì?

| Scenario | Day 08 calls | Day 09 calls |
|---------|-------------|-------------|
| Simple query | 1 LLM call | 3 LLM calls (Supervisor -> Worker -> Sinh kết quả) |
| Complex query | 1 LLM call | 4+ LLM calls (Phải gọi MCP) |
| MCP tool call | N/A | 1-2 calls mỗi lần dùng tool |

**Nhận xét về cost-benefit:**
Đánh đổi rất rõ rệt: Day 09 sẽ tốn tiền API (Token) nhiều hơn gấp 3-4 lần và tốn thời gian chờ lâu hơn, nhưng chất lượng độ chính xác logic được bảo đảm cao nhất.
_________________

---

## 6. Kết luận

> **Multi-agent tốt hơn single agent ở điểm nào?**

1. Độ đáng tin cậy cao, chia nhỏ vấn đề ra xử lý không bị lẫn lộn context.
2. Dễ Debug và dễ kết nối Tool qua hệ thống quản lý Trace/MCP.

> **Multi-agent kém hơn hoặc không khác biệt ở điểm nào?**

1. Kém hơn Single RAG Day 8 ở mặt tốc độ (Latency) do "overhead" khi điều hướng. Đôi khi tác vụ quá đơn giản như "Chính sách quy định là gì" nhưng vẫn phải đi qua vài vòng LLM.

> **Khi nào KHÔNG nên dùng multi-agent?**

Khi ứng dụng làm ra chủ yếu phục vụ các câu hỏi siêu cơ bản (FAQ dạng tĩnh) cần phản hồi < 2 giây.
_________________

> **Nếu tiếp tục phát triển hệ thống này, nhóm sẽ thêm gì?**

Tích hợp một thuật toán Hybrid vào Supervisor để nó quyết định "Câu hỏi này siêu dễ, không cần qua Graph Agent, trả lời luôn ngay lập tức" (nhằm tiết kiệm Token & Time xử lý các câu nhảm).
_________________
