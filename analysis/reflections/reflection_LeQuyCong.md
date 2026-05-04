# Individual Reflection — Lab 18

**Tên:** Lê Quý Công  
**Module phụ trách:** M1

---

## 1. Đóng góp kỹ thuật

- Module đã implement: Module 1 — Advanced Chunking Strategies cho hệ thống Production RAG.
- Các hàm/class chính đã viết:
  - `Chunk`: dataclass biểu diễn một chunk văn bản kèm metadata và `parent_id`.
  - `chunk_semantic`: chia văn bản theo mức độ tương đồng ngữ nghĩa giữa các câu bằng sentence embeddings.
  - `chunk_hierarchical`: tạo cấu trúc parent-child chunking, trong đó child chunk phục vụ truy xuất chính xác và parent chunk giữ ngữ cảnh rộng hơn.
  - `chunk_structure_aware`: chia tài liệu Markdown theo cấu trúc heading, giữ thông tin section trong metadata.
  - `compare_strategies`: so sánh basic, semantic, hierarchical và structure-aware chunking theo số lượng chunk và độ dài chunk.
- Số tests pass: 13/13

## 2. Kiến thức học được

- Khái niệm mới nhất: Các chiến lược chunking nâng cao trong RAG, đặc biệt là hierarchical chunking và structure-aware chunking.
- Điều bất ngờ nhất: Chunking không chỉ là cắt văn bản theo độ dài cố định; cách chia chunk ảnh hưởng trực tiếp đến chất lượng retrieval, độ đầy đủ ngữ cảnh và khả năng trả lời chính xác của hệ thống RAG.
- Kết nối với bài giảng (slide nào): Phần Production RAG về data preprocessing/chunking, retrieval quality và trade-off giữa precision với context preservation.

## 3. Khó khăn & Cách giải quyết

- Khó khăn lớn nhất: Thiết kế chunking sao cho vừa không làm mất ngữ cảnh, vừa tạo ra các đoạn đủ nhỏ để retrieval hiệu quả.
- Cách giải quyết: Implement nhiều chiến lược để so sánh: semantic chunking để nhóm câu cùng chủ đề, hierarchical chunking để liên kết child với parent, và structure-aware chunking để tận dụng cấu trúc Markdown của tài liệu.
- Thời gian debug: Khoảng 1–2 giờ, chủ yếu để kiểm tra metadata, `parent_id` của hierarchical chunks và đảm bảo các test case cho từng chiến lược đều pass.

## 4. Nếu làm lại

- Sẽ làm khác điều gì: Sẽ benchmark chất lượng retrieval sớm hơn trên tập câu hỏi thật, thay vì chỉ kiểm tra số lượng chunk và cấu trúc metadata.
- Module nào muốn thử tiếp: Module 2 — Hybrid Search, vì đây là bước trực tiếp sử dụng kết quả chunking để cải thiện khả năng truy xuất tài liệu.

## 5. Tự đánh giá

| Tiêu chí | Tự chấm (1-5) |
|----------|---------------|
| Hiểu bài giảng | 5 |
| Code quality | 4 |
| Teamwork | 4 |
| Problem solving | 4 |
