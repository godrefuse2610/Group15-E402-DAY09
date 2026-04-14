# Báo Cáo Cá Nhân — Lab Day 09: Multi-Agent Orchestration

**Họ và tên:** Nguyễn Công Nhật Tân (MSSV: 2A202600141)  
**Vai trò trong nhóm:** Trace & Docs Owner (Eval Owner)  
**Ngày nộp:** 14/4/2026  

---

## 1. Tôi phụ trách phần nào?

**Module/file tôi chịu trách nhiệm:**
- File chính: `eval_trace.py`, Day 8 `eval.py`, `docs/single_vs_multi_comparison.md`.
- Functions tôi implement: `analyze_traces()`, `compare_single_vs_multi()` trong `eval_trace.py`; viết thêm logic lưu `day08_results.json` cho codebase Day 08 và chèn `load_dotenv()` vào Pipeline.

**Cách công việc của tôi kết nối với phần của thành viên khác:**
Tôi là chốt chặn cuối cùng kiểm tra chất lượng. Các bạn dev (Worker/Supervisor Owners) viết code và xử lý logic luồng Agent; công việc của tôi là chạy test cases, thu gom lại các Trace log của họ thành `artifacts/traces` để tính toán Latency, Confidence và đo lường độ phân rã Context. Khi chạy Grading, tôi đã phát hiện và cứu nhóm một bàn thua trông thấy khi luồng Synthesis không gọi được LLM!

---

## 2. Tôi đã ra một quyết định kỹ thuật gì?

**Quyết định:** Viết bổ sung thêm một khối module tự động tính toán Time-Latency ở bên code Day 08 (file `day8/Group15-E402-DAY8/lab/eval.py`), và xuất kết quả ra file `day08_results.json` để Pipeline Day 09 (`eval_trace.py`) tự động load vào hàm `compare_single_vs_multi()`.

**Lý do:**
Ban đầu, biến `day08_baseline` trong file Eval của Day 09 chỉ là các con số Hard-Code. Để việc so sánh minh bạch và khách quan, tôi quyết định nâng cấp luôn code Day 08, ép nó xuất ra 1 file chuẩn format JSON có `latency_ms` và truyền thẳng absolute path của nó sang môi trường chạy Day 09.

**Trade-off đã chấp nhận:**
Mất một chút thời gian xử lý exception handling cũ của Lab Day 08. Đổi lại đồ thị báo cáo cực chuẩn xác số milliseconds.

**Bằng chứng từ code (cập nhật file eval_trace.py):**
```python
def compare_single_vs_multi(
    multi_traces_dir: str = "artifacts/traces",
    day08_results_file: Optional[str] = r"c:\Users\Admin\LAB\day8-9-10\day8\Group15-E402-DAY8\lab\results\day08_results.json",
):
```

---

## 3. Tôi đã sửa một lỗi gì?

**Lỗi siêu to khổng lồ:** Traces sinh ra từ file `eval_trace.py --grading` luôn bị `[SYNTHESIS ERROR] Không thể gọi LLM` dù API Key hoàn toàn hợp lệ, đe dọa điểm 0 ở phần 30 điểm Grading Questions!

**Symptom:**
Khi gọi `python eval_trace.py --grading`, toàn bộ 10 JSON output đều có answer lỗi `"Không thể gọi LLM. Kiểm tra API key trong .env."`. Tưởng do file config nhưng thực chất biến OS ảo hoàn toàn trống.

**Root cause:**
Lỗi do skeleton code mặc định từ lab không hề load biến môi trường! File `eval_trace.py` gọi vào `graph.py` và `synthesis.py` nhưng KHÔNG có bất kỳ hàm `load_dotenv()` nào ở root script, dẫn tới `os.getenv("OPENAI_API_KEY")` luôn bằng None.

**Cách sửa:**
Can thiệp sâu vào `eval_trace.py`, import và khởi chạy hàm `load_dotenv()` ngay ở dòng 20 trước khi script bắt đầu làm việc.

**Bằng chứng trước/sau:**
*Code sửa chữa vào top của eval_trace.py:*
```python
import argparse
from dotenv import load_dotenv
load_dotenv()
from datetime import datetime
```
*Kết quả Trace:* Từ `[SYNTHESIS ERROR]` đã chạy ra "Câu trả lời của pipeline...", LLM hoạt động thành công lấy được `conf=0.70`, mang lại điểm tối đa cho phần Grading Run!

---

## 4. Tôi tự đánh giá đóng góp của mình

**Tôi làm tốt nhất ở điểm nào?**
Debugging xuất thần và bao quát toàn hệ thống! Tôi đã giải quyết tận gốc 2 bug trí mạng của hệ thống: 1 là vòng lặp tải Model liên tục ở Worker Retrieval bằng cache `HF_HUB_OFFLINE`, 2 là lỗi mất cờ `.env` khi chạy Grading Test. 

**Tôi làm chưa tốt hoặc còn yếu ở điểm nào?**
Do phải chờ các bạn hoàn thành Pipeline mới có dữ liệu chốt để eval, khoảng đầu Lab phải chờ khá thụ động.

**Nhóm phụ thuộc vào tôi ở đâu?**
Nếu không có khâu Eval của tôi chặn lại và đọc log JSON, toàn nhóm sẽ submit 1 tệp `grading_run.jsonl` chứa 100% lỗi Error trong đáp án và mất toàn bộ 30 điểm Grading Questions từ thầy.

**Phần tôi phụ thuộc vào thành viên khác:**
Tôi cần file luồng routing từ Supervisor chạy chuẩn thì logic tổng mới test và chạy được các artifact comparison report.

---

## 5. Nếu có thêm 2 giờ, tôi sẽ làm gì?

Tôi sẽ tích hợp **LLM-as-a-judge** thẳng vào `eval_trace.py`. Vì trace của lệnh grading chạy cực kỳ chi tiết, tôi có thể tự động đọc output answer để so khớp thang điểm `Faithfulness` thay vì chỉ đo millisecond. Tôi muốn chạy thêm 1 vòng Eval tự động để lấy đồ thị Radar so với Ground Truth.
