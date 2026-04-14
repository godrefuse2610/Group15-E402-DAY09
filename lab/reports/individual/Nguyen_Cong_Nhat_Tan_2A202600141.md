# Báo Cáo Cá Nhân — Lab Day 09: Multi-Agent Orchestration

**Họ và tên:** Nguyễn Công Nhật Tân 
**Vai trò trong nhóm:** Trace & Docs Owner (Trace Owner)  
**Ngày nộp:** 14/4/2026  

---

## 1. Tôi phụ trách phần nào?

**Module/file tôi chịu trách nhiệm:**
- File chính: `eval_trace.py`, Day 8 `eval.py`, `docs/single_vs_multi_comparison.md`.
- Functions tôi implement: `analyze_traces()`, `compare_single_vs_multi()` trong `eval_trace.py`; viết thêm logic lưu `day08_results.json` cho codebase Day 08.

**Cách công việc của tôi kết nối với phần của thành viên khác:**
Tôi là chốt chặn cuối cùng kiểm tra chất lượng. Các bạn dev (Worker/Supervisor Owners) viết code và xử lý logic luồng Agent; công việc của tôi là chạy 15 test cases, thu gom lại các Trace log của họ thành `artifacts/traces` để tính toán Latency, Confidence và đo lường độ phân rã Context. Từ đó chứng minh hệ thống Multi-Agent Day 09 có độ an toàn cao hơn Day 08. Nếu file JSON Trace của họ làm sai Contract, script Eval của tôi sẽ báo lỗi.

---

## 2. Tôi đã ra một quyết định kỹ thuật gì?

**Quyết định:** Viết bổ sung thêm một khối module tự động tính toán Time-Latency ở bên code Day 08 (file `day8/Group15-E402-DAY8/lab/eval.py`), và xuất kết quả ra file `day08_results.json` để Pipeline Day 09 (`eval_trace.py`) tự động load vào hàm `compare_single_vs_multi()`.

**Lý do:**
Ban đầu, biến `day08_baseline` trong file Eval của Day 09 chỉ là các con số Hard-Code (hoặc "N/A"). Để việc so sánh hoàn toàn minh bạch và khoa học 100%, tôi quyết định không điền số tay mà nâng cấp luôn code Day 08, ép nó chạy ra 1 file chuẩn format JSON có `latency_ms` và truyền thẳng absolute path của nó sang môi trường chạy Day 09.

**Trade-off đã chấp nhận:**
Khá mất thời gian vì phải đọc lại code `run_scorecard` xa xưa ở Lab Day 08 và chèn `time.time()` vào khối Exception Handling. Đổi lại, file báo cáo `single_vs_multi_comparison.md` có tính tự động cao.

**Bằng chứng từ code:**
```python
# Cập nhật eval.py bên Day 08
latency_ms = int((time.time() - start_t) * 1000)
day08_json = {
    "total_questions": len(baseline_results),
    "avg_confidence": 0.0, 
    "avg_latency_ms": int(avg_lat)
...
# Cập nhật eval_trace.py bên Day 09
def compare_single_vs_multi(
    day08_results_file: Optional[str] = r"c:\Users\Admin\LAB\day8-9-10\day8\Group15-E402-DAY8\lab\results\day08_results.json"
):
```

---

## 3. Tôi đã sửa một lỗi gì?

**Lỗi:** Script Evaluation bị Treo vô hạn và Timeout (Crash `KeyboardInterrupt`) khi chạy tới các câu hỏi gọi Retrieval.

**Symptom (pipeline làm gì sai?):**
Khi tôi gõ `python eval_trace.py`, hệ thống in ra 1 câu rồi Loading Weights 100MB của `SentenceTransformer`. Tới câu thứ 3, hệ thống khựng lại, báo lỗi kết nối `ssl.py` và sập do Timeout.

**Root cause:**
Lỗi nằm ở module `workers/retrieval.py` khi bạn dev ném dòng `model = SentenceTransformer(...)` vào **bên trong** hàm. Do vòng lặp của Multi-Agent, cứ mỗi khi gọi Search_KB trên MCP hoặc tới Node Retrieval, LLM lại tải lại nguyên một model Offline 100MB và bắt API HuggingFace gọi check_update mạng. Do Rate limits, HuggingFace từ chối kết nối dẫn tới treo máy.

**Cách sửa:**
Rút `model` ra khỏi scope local, định nghĩa nó bằng một biến toàn cục `_STATIC_HF_MODEL` tải lười (Lazy Loaded), và cấm HuggingFace call mạng bằng `os.environ["HF_HUB_OFFLINE"] = "1"`.

**Bằng chứng cách sửa (Workers/Retrieval.py):**
```python
        import os
        os.environ["HF_HUB_OFFLINE"] = "1" # Khoá check update mạng, dùng local cache Day 8
        from sentence_transformers import SentenceTransformer
        global _STATIC_HF_MODEL
        if _STATIC_HF_MODEL is None:
            _STATIC_HF_MODEL = SentenceTransformer("all-MiniLM-L6-v2") # Load đúng 1 lần duy nhất
```
*Kết quả:* Tốc độ phản hồi câu 2 đến 15 giảm xuống chớp nhoáng chỉ còn ~15 ms (thay vì 15,000 ms treo máy).

*(Chú thích: Tôi cũng sửa thêm một lỗi `UnicodeDecodeError` do Windows không đọc được utf-8 khi gom log trong file JSON)*.

---

## 4. Tôi tự đánh giá đóng góp của mình

**Tôi làm tốt nhất ở điểm nào?**
Debugging cực mạnh về mặt hiệu năng. Việc xử lý thành công `HF_HUB_OFFLINE` cứu Group khỏi việc bị trừ điểm Timeout chạy Lab, đồng thời hoàn thành file Comparison rất chi tiết và khách quan với số đo ms chuẩn.

**Tôi làm chưa tốt hoặc còn yếu ở điểm nào?**
Do phải chờ các bạn hoàn thành Pipeline mới có dữ liệu chốt để eval, đoạn đầu phiên Lab tôi chưa thể debug chung được sâu vào prompt mà chỉ focus được vào Trace File Format.

**Nhóm phụ thuộc vào tôi ở đâu?**
Các bạn bắt buộc phải có script rút Logs của tôi để xuất ra các bản ghi Artifacts JSON làm bằng chứng chấm điểm nộp cho thầy.

**Phần tôi phụ thuộc vào thành viên khác:**
Tôi cần file `eval_trace` được fix logic routing thì mới sinh ra đồ thị test được trọn vẹn 15 câu (không bị rớt) để chạy báo cáo.

---

## 5. Nếu có thêm 2 giờ, tôi sẽ làm gì?

Tôi sẽ tích hợp **LLM-as-a-judge** thẳng vào `eval_trace.py`. Vì trace của `gq09` cho thấy `hitl_triggered=True` và `confidence=0.41`, chứng tỏ AI đã phản ứng đúng với "Unknow error code", nhưng Eval báo cáo đơn thuần vẫn chưa có thang điểm `Faithfulness` để chấm câu `gq09` đó. Tôi muốn viết thêm bước chấm tự động dựa trên output trả về so với Ground Truth.
