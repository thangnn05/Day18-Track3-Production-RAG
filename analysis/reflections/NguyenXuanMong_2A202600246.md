# Individual Reflection — Lab 18

**Tên:** Nguyen Xuan Mong  
**Module phụ trách:** M2 — Hybrid Search

---

## 1. Đóng góp kỹ thuật

- Module đã implement: `src/m2_search.py` — Hybrid Search kết hợp BM25 tiếng Việt, Dense Vector Search và Reciprocal Rank Fusion.
- Các hàm/class chính đã viết:
  - `segment_vietnamese()`: tách từ tiếng Việt bằng `underthesea.word_tokenize(text, format="text")`, có fallback khi môi trường chưa cài package.
  - `BM25Search.index()`: xây dựng BM25 index từ các text chunks sau khi segment và tokenize.
  - `BM25Search.search()`: search theo query bằng BM25, trả về danh sách `SearchResult` với `method="bm25"`.
  - `_fallback_scores()`: BM25 scoring fallback thủ công nếu thiếu thư viện `rank_bm25`.
  - `DenseSearch.index()`: encode chunks bằng embedding model `BAAI/bge-m3`, tạo `PointStruct`, upload vào Qdrant.
  - `DenseSearch.search()`: encode query và search vector trong Qdrant, hỗ trợ cả `client.search()` và `query_points()` tùy version.
  - `reciprocal_rank_fusion()`: merge kết quả BM25 và dense search bằng công thức RRF.
- Số tests pass: 5/5 (`pytest tests/test_m2.py`)

## 2. Kiến thức học được

- Khái niệm mới nhất: Hybrid Search trong RAG — kết hợp lexical search và semantic search để tăng chất lượng retrieval.
- Điều bất ngờ nhất: BM25 và dense retrieval mạnh ở hai hướng khác nhau. BM25 bắt keyword chính xác tốt, còn dense search bắt ý nghĩa/ngữ cảnh tốt. Khi dùng RRF để fusion, hệ thống ổn định hơn so với chỉ dùng một phương pháp.
- Kết nối với bài giảng: phần Production RAG / Retrieval Optimization, đặc biệt các kỹ thuật BM25, embedding search, vector database và reranking/fusion.

## 3. Khó khăn & Cách giải quyết

- Khó khăn lớn nhất: đảm bảo code chạy được trong nhiều môi trường khác nhau, vì một số thư viện như `underthesea`, `rank_bm25`, hoặc API Qdrant có thể khác version hoặc chưa được cài.
- Cách giải quyết: thêm fallback cho Vietnamese segmentation và BM25 scoring; với Qdrant, hỗ trợ cả `client.search()` và `client.query_points()` để tương thích nhiều version.
- Thời gian debug: khoảng 20–30 phút, chủ yếu để kiểm tra test case, xử lý dependency fallback và đảm bảo output đúng format `SearchResult`.

## 4. Nếu làm lại

- Sẽ làm khác điều gì: chuẩn bị file requirements rõ hơn cho các dependency cần thiết như `underthesea`, `rank_bm25`, `sentence-transformers`, `qdrant-client` để tránh lệch môi trường.
- Module nào muốn thử tiếp: M3 — Reranking, vì sau khi hybrid search lấy được candidates, reranker có thể cải thiện thứ tự kết quả cuối cùng trước khi đưa vào LLM.

## 5. Tự đánh giá

| Tiêu chí | Tự chấm (1-5) |
|----------|---------------|
| Hiểu bài giảng | 5 |
| Code quality | 5 |
| Teamwork | 4 |
| Problem solving | 5 |
