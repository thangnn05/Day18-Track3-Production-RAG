# Individual Reflection — Lab 18

**Tên:** Nguyễn Ngọc Thắng
**Module phụ trách:** M3 — Reranking

---

## 1. Đóng góp kỹ thuật

- **Module đã implement:** M3 — Reranking (`src/m3_rerank.py`)
- **Các hàm/class chính đã viết:**
  - `CrossEncoderReranker._load_model()` — lazy-load `BAAI/bge-reranker-v2-m3` qua
    `sentence_transformers.CrossEncoder` (chỉ load khi rerank lần đầu, tránh import-time cost).
  - `CrossEncoderReranker.rerank(query, documents, top_k)` — build pairs `(query, doc)`,
    chạy `model.predict()`, sort theo score giảm dần, trả về `RerankResult` với
    `original_score` (giữ điểm hybrid gốc), `rerank_score`, và `rank` để dễ debug.
  - `FlashrankReranker` (optional) — bọc `flashrank.Ranker` để có alternative
    nhẹ hơn (~5ms) cho production có ngân sách latency thấp; gracefully `return []`
    nếu chưa cài flashrank.
  - `benchmark_reranker()` — đo latency `n_runs` lần với `time.perf_counter()`,
    có **warm-up run** ngoài vòng đo để loại trừ cold-start (đây là điểm mình
    thấy nhiều baseline ở net làm sai, model load ~3s sẽ dominate average).
- **Số tests pass:** 5/5 (`tests/test_m3.py`)

## 2. Kiến thức học được

- **Khái niệm mới nhất:** Khác biệt giữa **bi-encoder** (M2 dense search dùng
  `bge-m3`) và **cross-encoder** (M3 dùng `bge-reranker-v2-m3`). Bi-encoder
  encode query và doc độc lập → một dot product → nhanh nhưng mất interaction
  giữa từng token query với từng token doc. Cross-encoder feed `(query, doc)`
  cùng lúc qua transformer → full self-attention giữa query↔doc → score chính
  xác hơn nhưng O(N) forward passes thay vì O(1). Vì vậy pattern chuẩn:
  retrieve 20 candidates rẻ bằng bi-encoder rồi rerank xuống 3 bằng cross-encoder.
- **Điều bất ngờ nhất:** Cross-encoder score **không phải** xác suất —
  raw logit, có thể âm, không sort được giữa các queries khác nhau. Chỉ
  có ý nghĩa relative trong cùng 1 query. Điều này có ý nghĩa khi muốn set
  threshold "min relevance" — phải calibrate cho mỗi domain.
- **Kết nối với bài giảng:** Slide phần "two-stage retrieval" + slide về
  RAGAS `context_precision` — module M3 chính là cách tốt nhất để kéo
  metric này lên: thay vì giữ 20 chunks (nhiều noise) thì giữ 3 chunks
  có ranking cross-encoder cao nhất → faithfulness và answer_relevancy
  cũng tăng theo vì LLM ít bị distract.

## 3. Khó khăn & Cách giải quyết

- **Khó khăn lớn nhất:** Lần đầu chạy benchmark, average latency lên ~800ms/query
  trong khi Flashrank quảng cáo <5ms — mình tưởng implement sai. Sau khi đào
  log thì hiểu ra: model load (`SentenceTransformer.__init__` + tải weights ~2GB)
  bị tính vào lần đo đầu tiên, làm lệch trung bình. Steady-state thực ra
  chỉ ~50ms/query trên CPU.
- **Cách giải quyết:** Thêm 1 warm-up call `reranker.rerank(query, documents)`
  trước vòng đo trong `benchmark_reranker()`. Ngoài ra tách `_load_model()`
  ra riêng và lazy — pipeline.py có thể construct `CrossEncoderReranker()`
  rất nhẹ, chỉ trả tiền khi thực sự rerank.
- **Thời gian debug:** ~25 phút (chủ yếu là kiên nhẫn chờ model download lần đầu
  để xác nhận giả thuyết warm-up cost).

## 4. Nếu làm lại

- **Sẽ làm khác điều gì:**
  - Cache cross-encoder scores theo `(query_hash, doc_hash)` — trong benchmark
    nội bộ thấy nhiều query repeat trong test set, có thể tiết kiệm 30-40%
    inference cost.
  - Batch nhiều queries vào 1 `model.predict()` call (batch_size=32) thay vì
    1 query × N docs — throughput sẽ tốt hơn cho production khi nhiều user
    cùng lúc.
- **Module nào muốn thử tiếp:** M5 (Enrichment) — đặc biệt là **Contextual
  Prepend** style Anthropic (-49% retrieval failure). Mình tin combo
  enrichment + rerank sẽ cộng dồn (M3 cải thiện precision, M5 cải thiện
  recall) thay vì overlap.

## 5. Tự đánh giá

| Tiêu chí | Tự chấm (1-5) |
|----------|---------------|
| Hiểu bài giảng | 4 |
| Code quality | 4 |
| Teamwork | 4 |
| Problem solving | 4 |
