# Individual Reflection — Lab 18

**Tên:** Trần Minh Toàn  
**Module phụ trách:** M4 — RAGAS Evaluation & Failure Analysis

---

## 1. Đóng góp kỹ thuật

- **Phần mình phụ trách:** Xây dựng module đánh giá chất lượng RAG bằng RAGAS, gồm 4 metrics chuẩn và cơ chế phân tích lỗi theo Diagnostic Tree.
- **Các thành phần chính đã triển khai:**
  - `EvalResult`: dataclass lưu kết quả theo từng câu hỏi.
  - `load_test_set()`: đọc tập test từ `test_set.json`.
  - `evaluate_ragas()`: chạy đánh giá RAGAS, tổng hợp điểm trung bình, xử lý lỗi runtime và NaN-safe.
  - `failure_analysis()`: chọn bottom-N case yếu nhất, xác định lỗi chính, gợi ý hướng sửa.
  - `save_report()`: ghi report JSON để nhóm dùng cho so sánh baseline vs production.
  - `_get_ragas_llm()`: khởi tạo LLM backend bằng `llm_factory("gpt-4o-mini", client=OpenAI(...))`.
  - `_get_ragas_embeddings()`: dùng `langchain_openai.OpenAIEmbeddings` để tương thích interface metric Answer Relevancy yêu cầu.
- **Trạng thái kiểm thử:** test module M4 pass 4/4.

## 2. Kiến thức học được

- **Hiểu sâu hơn về 4 metric RAGAS:**
  - **Faithfulness:** mức độ trung thực của câu trả lời so với context.
  - **Answer Relevancy:** mức độ trả lời đúng intent câu hỏi.
  - **Context Precision:** tỉ lệ context truy xuất thực sự liên quan.
  - **Context Recall:** mức độ bao phủ ground truth bởi context truy xuất.
- **Bài học quan trọng:** cải thiện retrieval chưa đủ để hệ thống tốt toàn diện. Trong kết quả production, `context_precision` và `context_recall` tăng rõ, nhưng `faithfulness` vẫn có thể giảm nếu bước generation chưa được ràng buộc chặt vào evidence.
- **Liên hệ bài giảng:** đúng với framework phân tích lỗi 3 tầng: Output -> Context -> Query.

## 3. Khó khăn và cách giải quyết

- **Vấn đề 1:** Lỗi `All metrics must be initialised metric objects`.
- **Cách xử lý:** dùng metric instances từ `ragas.metrics` (đúng kiểu `Metric`) thay vì `ragas.metrics.collections.*` cho phiên bản hiện tại của `evaluate()`.

- **Vấn đề 2:** Lỗi `HuggingFaceEmbeddings object has no attribute embed_query` khi chạy Answer Relevancy.
- **Cách xử lý:** chuyển sang `langchain_openai.OpenAIEmbeddings` để có đủ `embed_query` và `embed_documents`.

- **Vấn đề 3:** `llm_factory` yêu cầu `client` ở bản RAGAS mới.
- **Cách xử lý:** tạo `OpenAI(api_key=...)` và truyền vào `llm_factory`.

- **Kết quả đạt được:** pipeline chạy end-to-end bằng `python main.py`, report sinh ổn định, đủ dữ liệu aggregate để so sánh baseline và production.

## 4. Nếu làm lại

- Pin version thư viện từ đầu (RAGAS, LangChain, OpenAI SDK) để tránh API drift giữa lúc làm.
- Viết checklist tương thích ngay tuần đầu (model I/O, metric object type, embeddings interface).
- Bổ sung logging chi tiết hơn cho failure report (lưu đầy đủ question, answer, retrieved contexts) để phân tích case-level chính xác hơn.

## 5. Tự đánh giá

| Tiêu chí | Tự chấm (1-5) |
|----------|---------------|
| Hiểu bài giảng | 4 |
| Code quality | 4 |
| Teamwork | 4 |
| Problem solving | 5 |

> **Ghi chú:** Mình chấm Problem solving 5/5 vì đã tự trace và xử lý được chuỗi lỗi tương thích thư viện để pipeline chạy lại ổn định. Mình giữ Code quality ở mức 4/5 vì failure artifact hiện vẫn thiếu một số trường text để phân tích case cụ thể sâu hơn.
