# Individual Reflection — Lab 18

**Tên:** Trần Minh Toàn 
**Module phụ trách:** M4 — RAGAS Evaluation & Failure Analysis

---

## 1. Đóng góp kỹ thuật

- **Module đã implement:** Module 4 — Hệ thống đánh giá RAG tự động bằng RAGAS với 4 metrics và phân tích failure theo Diagnostic Tree.
- **Các hàm/class chính đã viết:**
  - `EvalResult`: dataclass lưu kết quả đánh giá per-question (faithfulness, answer_relevancy, context_precision, context_recall).
  - `load_test_set()`: load 20 câu hỏi + ground truth từ `test_set.json`.
  - `evaluate_ragas()`: chạy RAGAS evaluate với LLM backend (`gpt-4o-mini`) + embedding backend (`nomic-embed-text-v1.5`), NaN-safe aggregation.
  - `failure_analysis()`: sort bottom-N questions theo average score, map từng failure sang Diagnostic Tree với root cause và suggested fix.
  - `save_report()`: xuất JSON report với aggregate scores + failure list.
  - `_get_ragas_llm()`: khởi tạo LLM backend dùng `llm_factory` (RAGAS API mới nhất).
  - `_get_ragas_embeddings()`: khởi tạo embedding backend với fallback chain.
- **Số tests pass:** 4/4

## 2. Kiến thức học được

- **Khái niệm mới nhất:** 4 metrics của RAGAS và ý nghĩa từng cái:
  - **Faithfulness** — LLM có hallucinate so với context không? (baseline: 0.82 — cao vì answer = chunk gốc)
  - **Answer Relevancy** — Answer có trả lời đúng câu hỏi không? (baseline: 0.33 — thấp vì không có LLM generation)
  - **Context Precision** — Trong các chunks retrieved, bao nhiêu % thực sự liên quan? (baseline: 0.43)
  - **Context Recall** — Ground truth có được cover bởi retrieved context không? (baseline: 0.35)
- **Điều bất ngờ nhất:** Faithfulness cao (0.82) không có nghĩa là hệ thống tốt — đây là "false positive" vì baseline trả thẳng chunk gốc làm answer, không qua LLM. Metric quan trọng hơn để cải thiện là Answer Relevancy (0.33) và Context Recall (0.35).
- **Kết nối với bài giảng:** Phần RAG Evaluation — slide về RAGAS metrics, Diagnostic Tree phân tích failure theo 3 tầng: Output → Context → Query.

## 3. Khó khăn & Cách giải quyết

- **Khó khăn lớn nhất:** RAGAS 0.4.x thay đổi API hoàn toàn so với docs cũ — `LangchainLLMWrapper`, `LangchainEmbeddingsWrapper`, `from ragas.metrics import ...` đều deprecated với warning. Nếu để nguyên thì code chạy được nhưng sẽ bị break ở version tiếp theo.
- **Cách giải quyết:** Implement 2 tầng: thử API mới (`llm_factory`, `ragas.metrics.collections`) trước, nếu fail thì fallback về API cũ. Đảm bảo backward compatible với cả RAGAS 0.3 và 0.4.
- **Thời gian debug:** ~45 phút để trace deprecation warnings và tìm API mới trong source code RAGAS; thêm ~20 phút để model `nomic-embed-text-v1.5` download xong (547MB) và xử lý lỗi thiếu `einops`.

## 4. Nếu làm lại

- **Sẽ làm khác điều gì:** Pin version RAGAS ngay từ đầu (`ragas==0.3.x`) thay vì dùng latest, tránh mất thời gian xử lý breaking changes. Hoặc migrate hẳn sang API mới với `@experiment` decorator thay vì `evaluate()`.
- **Module nào muốn thử tiếp:** M5 Enrichment — đặc biệt là kỹ thuật **HyDE (Hypothetical Document Embedding)** và **Contextual Prepend** theo paper của Anthropic. Muốn đo xem enrichment thực sự cải thiện context_recall bao nhiêu điểm.

## 5. Tự đánh giá

| Tiêu chí | Tự chấm (1-5) |
|----------|---------------|
| Hiểu bài giảng | 4 |
| Code quality | 4 |
| Teamwork | 4 |
| Problem solving | 5 |

> **Ghi chú:** Trừ 1 điểm Code quality vì vẫn còn warning từ thư viện `SwigPy` (PyArrow internal) không xử lý được. Problem solving 5/5 vì tự debug được API deprecation issue và implement fallback chain hoạt động ổn định.
