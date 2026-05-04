# Group Report — Lab 18: Production RAG

**Nhóm:** E4 
**Ngày:** 04/05/2026

## Thành viên & Phân công

| Tên | Module | Hoàn thành | Tests pass |
|-----|--------|-----------|-----------|
| Lê Quý Công | M1: Chunking | ☑ | 13/13 |
| Nguyễn Xuân Mong | M2: Hybrid Search | ☑ | 5/5 |
| Nguyễn Ngọc Thắng | M3: Reranking | ☑ | 5/5 |
| Trần Minh Toàn | M4: Evaluation | ☑ | 4/4 |

> **M5 (Enrichment):** được implement chung bởi cả nhóm (commit `53b3677`), tích hợp vào pipeline bởi Nguyễn Ngọc Thắng.

## Kết quả RAGAS

| Metric | Naive | Production | Δ |
|--------|-------|-----------|---|
| Faithfulness | 0.8224 | 0.6375 | −0.1849 |
| Answer Relevancy | 0.3321 | 0.3419 | +0.0098 |
| Context Precision | 0.4292 | 0.8000 | +0.3708 |
| Context Recall | 0.3500 | 0.6700 | +0.3200 |

## Key Findings

1. **Biggest improvement:** Context Precision (+0.3708) và Context Recall (+0.3200). Nhờ Hybrid Search (M2) kết hợp BM25 + Dense Search qua RRF, và Cross-Encoder Reranking (M3) lọc top-3 candidates chính xác nhất, hệ thống truy xuất đúng context hơn hẳn so với naive baseline.

2. **Biggest challenge:** Faithfulness giảm −0.1849 (từ 0.82 → 0.64). Dù retrieval cải thiện rõ rệt, bước generation (LLM) lại hallucinate nhiều hơn. Nguyên nhân: prompt chưa ràng buộc đủ chặt để LLM chỉ dựa vào context được truy xuất. Đây là bài học quan trọng — cải thiện retrieval chưa đủ, cần kiểm soát cả generation.

3. **Surprise finding:** Cross-encoder score không phải xác suất — raw logit có thể âm, chỉ có ý nghĩa relative trong cùng 1 query (Thắng — M3). BM25 và Dense search mạnh ở hai hướng khác nhau: BM25 bắt keyword chính xác, Dense bắt ngữ nghĩa/ngữ cảnh — RRF fusion kết hợp tốt cả hai ưu điểm (Mong — M2).

## Tổng hợp khó khăn kỹ thuật

| Thành viên | Module | Khó khăn chính | Cách giải quyết | Thời gian debug |
|-----------|--------|---------------|-----------------|----------------|
| Lê Quý Công | M1 | Cân bằng kích thước chunk vs. giữ ngữ cảnh | Implement nhiều chiến lược để so sánh (semantic, hierarchical, structure-aware) | 1–2 giờ |
| Nguyễn Xuân Mong | M2 | Dependency khác version/thiếu thư viện (`underthesea`, `rank_bm25`, Qdrant API) | Thêm fallback cho Vietnamese segmentation, BM25 scoring; hỗ trợ cả `client.search()` và `client.query_points()` | 20–30 phút |
| Nguyễn Ngọc Thắng | M3 | Model load ~2GB lần đầu dominate latency benchmark | Thêm warm-up call trước vòng đo; tách `_load_model()` lazy loading | ~25 phút |
| Trần Minh Toàn | M4 | Chuỗi lỗi tương thích RAGAS/LangChain/OpenAI SDK | Chuyển sang metric instances đúng kiểu, dùng `langchain_openai.OpenAIEmbeddings`, tạo `OpenAI(api_key=...)` cho `llm_factory` | — |

## Tự đánh giá nhóm

| Thành viên | Hiểu bài giảng | Code quality | Teamwork | Problem solving |
|-----------|---------------|-------------|----------|----------------|
| Lê Quý Công | 5 | 4 | 4 | 4 |
| Nguyễn Xuân Mong | 5 | 5 | 4 | 5 |
| Nguyễn Ngọc Thắng | 4 | 4 | 4 | 4 |
| Trần Minh Toàn | 4 | 4 | 4 | 5 |
| **Trung bình** | **4.5** | **4.25** | **4.0** | **4.5** |

## Presentation Notes (5 phút)

1. **RAGAS scores (naive vs production):**
   - Context Precision: 0.43 → 0.80 (+87% improvement) — nhờ Hybrid Search + Reranking
   - Context Recall: 0.35 → 0.67 (+91% improvement) — retrieval tìm được nhiều context liên quan hơn
   - Faithfulness: 0.82 → 0.64 (−22% regression) — cần tối ưu prompt generation
   - Answer Relevancy: 0.33 → 0.34 (stable) — chưa cải thiện rõ, cần prompt tuning

2. **Biggest win — module nào, tại sao:**
   - M2 (Hybrid Search) + M3 (Reranking) là combo mạnh nhất: M2 tăng recall bằng fusion BM25+Dense, M3 tăng precision bằng cross-encoder lọc top-3. Context Precision tăng gần gấp đôi.

3. **Case study — 1 failure, Error Tree walkthrough:**
   - Case faithfulness=0.0: Output sai → LLM hallucinate thay vì dựa context → Context có nhưng prompt không ràng buộc → Fix: thêm guardrail "Trả lời CHỈ dựa trên context", lower temperature=0.

4. **Next optimization nếu có thêm 1 giờ:**
   - Tighten generation prompt: thêm instruction "Nếu không đủ thông tin trong context, trả lời 'Không đủ thông tin'" → fix faithfulness regression
   - Cache cross-encoder scores theo `(query_hash, doc_hash)` để tiết kiệm 30-40% inference cost
   - Bổ sung logging đầy đủ question/answer/context trong failure report để truy vết case-level
