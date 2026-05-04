# Failure Analysis — Lab 18: Production RAG

**Nhóm:** Production RAG Group  
**Thành viên:** → M1 · Lê Quý Công → M2 · Nguyễn Ngọc Thắng → M3 · Trần Minh Toàn → M4 · 

---

## RAGAS Scores

| Metric | Naive Baseline | Production | Δ |
|--------|---------------|------------|---|
| Faithfulness | 0.8224 | — | — |
| Answer Relevancy | 0.3321 | — | — |
| Context Precision | 0.4292 | — | — |
| Context Recall | 0.3500 | — | — |

> Scores được tính tự động bởi `naive_baseline.py` → `reports/naive_baseline_report.json`

---

## Bottom-5 Failures

Baseline sử dụng **paragraph chunking + dense-only search**, không có rerank và không có LLM generation (answer = top-1 retrieved chunk). Các failure chính dự kiến xảy ra do:
- Answer relevancy thấp: câu trả lời là đoạn văn nguyên bản, không phải câu trả lời có cấu trúc.
- Context recall thấp: dense search đơn thuần bỏ sót keyword-heavy queries.

### #1
- **Question:** Nghị định 13/2023/NĐ-CP về bảo vệ dữ liệu cá nhân được ban hành vào ngày tháng năm nào?
- **Expected:** Nghị định 13/2023/NĐ-CP được ban hành ngày 17 tháng 4 năm 2023.
- **Got:** _(top-1 retrieved chunk — thường là đoạn giới thiệu chung, không chứa ngày cụ thể)_
- **Worst metric:** context_recall
- **Error Tree:** Output sai → Context có chứa ngày không? → **Không** → Dense search bỏ sót chunk chứa số ngày → Query thiếu keyword số → BM25 hybrid sẽ giải quyết được
- **Root cause:** Dense embedding không nhạy với số cụ thể ("17", "4", "2023") — semantic search bỏ sót.
- **Suggested fix:** Thêm BM25 hybrid search (M2) để bắt keyword số chính xác.

### #2
- **Question:** Khi chủ thể dữ liệu yêu cầu hạn chế xử lý dữ liệu, việc hạn chế phải được thực hiện trong thời gian bao lâu?
- **Expected:** Việc hạn chế xử lý dữ liệu phải được thực hiện trong 72 giờ sau khi có yêu cầu của chủ thể dữ liệu.
- **Got:** _(chunk về quyền hạn chế — có thể đúng hoặc không chứa "72 giờ")_
- **Worst metric:** context_precision
- **Error Tree:** Output sai → Context đúng chủ đề nhưng thiếu "72 giờ"? → **Có** → Chunk quá lớn, "72 giờ" nằm ở chunk khác → Hierarchical chunking (M1) sẽ giúp
- **Root cause:** Paragraph chunking cắt không đúng ranh giới — số giờ cụ thể nằm ở chunk kề bên.
- **Suggested fix:** Hierarchical chunking (parent 2048 / child 256) giữ ngữ cảnh tốt hơn.

### #3
- **Question:** Số thuế GTGT còn phải nộp trong kỳ của Công ty DHA Surfaces theo tờ khai là bao nhiêu?
- **Expected:** Thuế GTGT còn phải nộp trong kỳ là 52.133.830 đồng.
- **Got:** _(chunk tổng quát về tờ khai GTGT — không chứa số cụ thể)_
- **Worst metric:** faithfulness
- **Error Tree:** Output sai → Answer có số tiền không? → **Không** → Context không chứa bảng số liệu → Markdown table không được chunk đúng cách
- **Root cause:** `chunk_basic()` không xử lý bảng Markdown — các dòng số liệu bị tách rời khỏi tiêu đề cột.
- **Suggested fix:** Thêm logic nhận diện bảng Markdown trong chunker (M1), giữ nguyên bảng trong một chunk.

### #4
- **Question:** Dữ liệu cá nhân nhạy cảm gồm những loại thông tin gì?
- **Expected:** _(danh sách 10 loại thông tin nhạy cảm)_
- **Got:** _(chỉ lấy được 1 chunk, bỏ sót phần liệt kê đầy đủ)_
- **Worst metric:** context_recall
- **Error Tree:** Output sai → Context có liệt kê đầy đủ không? → **Không** → Top-3 chunks không cover hết danh sách → Reranker (M3) chưa có để ưu tiên chunk đúng
- **Root cause:** Dense search trả về chunks chứa keyword "nhạy cảm" nhưng không phải chunk chứa definition đầy đủ.
- **Suggested fix:** Cross-encoder reranking (M3) để chọn đúng chunk định nghĩa.

### #5
- **Question:** Công ty CP DHA Surfaces có mã số thuế và kỳ tính thuế nào trên tờ khai GTGT?
- **Expected:** Mã số thuế 0106769437, kỳ tính thuế Quý 4 năm 2024.
- **Got:** _(chunk không chứa phần header tờ khai)_
- **Worst metric:** answer_relevancy
- **Error Tree:** Output sai → Answer có trả lời đúng câu hỏi không? → **Không** → Chunk trả về là phần nội dung giữa, không phải phần header → Metadata enrichment giúp tag loại document
- **Root cause:** Không có metadata về loại document/section — search không biết ưu tiên phần header tờ khai.
- **Suggested fix:** M5 enrichment: thêm metadata `doc_type`, `section` để filter/boost chunk header.

---

## Case Study (cho presentation)

**Question chọn phân tích:**  
*"Số thuế GTGT còn phải nộp trong kỳ của Công ty DHA Surfaces theo tờ khai là bao nhiêu?"*

**Error Tree walkthrough:**
1. **Output đúng?** → **Không** — answer là đoạn văn mô tả chung về GTGT, không có số tiền.
2. **Context đúng?** → **Không** — top-1 chunk không chứa bảng số liệu tờ khai.
3. **Search đúng?** → **Không** — dense search dùng semantic similarity; số `52.133.830` là out-of-distribution với embedding space.
4. **Fix ở bước:** M1 (giữ nguyên bảng Markdown) + M2 (thêm BM25 bắt keyword số) + M3 (rerank ưu tiên chunk tờ khai).

**Nếu có thêm 1 giờ, sẽ optimize:**
- Implement BM25 hybrid search (RRF fusion) — bắt được keyword số chính xác.
- Thêm table-aware chunking trong M1 — không tách bảng Markdown thành nhiều mảnh.
- Cross-encoder reranking với `ms-marco-MiniLM` — tăng context_precision lên đáng kể.
