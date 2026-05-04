# Failure Analysis — Lab 18: Production RAG

**Nhóm:** Production RAG Group  
**Thành viên:** M1 · Lê Quý Công → M2 · Nguyễn Xuân Mong → M3 · Nguyễn Ngọc Thắng → M4 · Trần Minh Toàn

---

## RAGAS Scores

| Metric | Naive Baseline | Production | Δ |
|--------|---------------|------------|---|
| Faithfulness | 0.8224 | 0.6375 | −0.1849 |
| Answer Relevancy | 0.3321 | 0.3419 | +0.0098 |
| Context Precision | 0.4292 | 0.8000 | +0.3708 |
| Context Recall | 0.3500 | 0.6700 | +0.3200 |

> Scores được tính tự động bởi `naive_baseline.py` và `main.py` → `reports/naive_baseline_report.json`, `reports/ragas_report.json`.

### Phân tích tổng quan

- **Retrieval cải thiện rõ rệt:** Context Precision (+0.3708) và Context Recall (+0.3200) tăng mạnh nhờ Hybrid Search (M2: BM25 + Dense + RRF) kết hợp Cross-Encoder Reranking (M3: `bge-reranker-v2-m3`).
- **Generation vẫn là điểm yếu:** Faithfulness giảm −0.1849, cho thấy LLM hallucinate nhiều hơn khi context phong phú hơn. Nguyên nhân: prompt chưa ràng buộc đủ chặt để LLM chỉ trả lời dựa trên evidence trong context.
- **Answer Relevancy ổn định:** gần như không thay đổi (+0.0098), cho thấy cải thiện retrieval chưa đủ — cần prompt tuning ở bước generation.

---

## Bottom-5 Failures

Dựa trên `reports/ragas_report.json`. Các case dưới đây là 5 câu hỏi có điểm trung bình RAGAS thấp nhất trong production pipeline.

> **Lưu ý:** Report JSON gốc bị thiếu trường `question`, `ground_truth`, `answer` (đều là `""`). Dưới đây đã bổ sung câu hỏi từ `test_set.json` dựa trên thứ tự index.

### #1
- **Worst metric:** faithfulness = 0.0
- **Scores:** faithfulness=0.0, answer_relevancy=0.0, context_precision=0.0, context_recall=0.0
- **Diagnosis:** LLM hallucinating — Tất cả 4 metric đều bằng 0, cho thấy pipeline có thể đã fail hoàn toàn (không truy xuất được context hoặc không sinh được answer).
- **Suggested fix:** Kiểm tra pipeline end-to-end cho case này; tighten prompt: "Trả lời CHỈ dựa trên context". Lower temperature to 0.
- **Root cause (Diagnostic Tree):**
  1. Output đúng? → **Không** (faithfulness=0.0)
  2. Context đúng? → **Không** (context_precision=0.0, context_recall=0.0)
  3. Query rewrite OK? → Có thể query quá cụ thể hoặc chunking không cover document phù hợp
  4. Fix ở bước: M1 (chunking coverage) + M2 (search recall) + M4 (prompt guardrail)

### #2
- **Worst metric:** faithfulness = 0.0
- **Scores:** faithfulness=0.0, answer_relevancy=0.0, context_precision=0.0, context_recall=0.0
- **Diagnosis:** LLM hallucinating — Tương tự case #1, pipeline fail toàn bộ.
- **Suggested fix:** Tương tự case #1. Thêm fallback: nếu retrieval trả về 0 kết quả, trả lời "Không đủ thông tin".

### #3
- **Worst metric:** faithfulness = 0.0
- **Scores:** faithfulness=0.0, answer_relevancy=0.0, context_precision=0.9999, context_recall=0.0
- **Diagnosis:** LLM hallucinating mặc dù context precision rất cao.
- **Insight:** Context truy xuất rất chính xác (precision ≈ 1.0) nhưng recall = 0 và faithfulness = 0. Điều này có nghĩa: retrieval tìm đúng document nhưng thiếu thông tin ground truth; LLM bịa thêm thay vì từ chối trả lời.
- **Suggested fix:** Tighten prompt: "Trả lời CHỈ dựa trên context. Nếu context không chứa đủ thông tin, trả lời: Không đủ thông tin trong ngữ cảnh." Lower temperature to 0.
- **Root cause:** Lỗi ở bước generation, không phải retrieval. Fix tại M4 (prompt template) + chunking coverage (M1).

### #4
- **Worst metric:** answer_relevancy = 0.0, faithfulness = 0.5
- **Scores:** faithfulness=0.5, answer_relevancy=0.0, context_precision=0.0, context_recall=0.5
- **Diagnosis:** Answer doesn't match question intent — câu trả lời không liên quan đến câu hỏi.
- **Insight:** faithfulness=0.5 cho thấy LLM có dựa vào context nhưng chỉ một nửa. context_recall=0.5 cho thấy retrieval tìm được một phần ground truth. Vấn đề chính: answer không đúng intent (answer_relevancy=0.0).
- **Suggested fix:** Improve prompt template: yêu cầu LLM restate lại câu hỏi trước khi trả lời. Ensure question context không bị mất khi truyền vào LLM.

### #5
- **Worst metric:** faithfulness = 0.0
- **Scores:** faithfulness=0.0, answer_relevancy=0.0, context_precision=0.8333, context_recall=0.4
- **Diagnosis:** LLM hallucinating dù context khá tốt.
- **Insight:** Context precision cao (0.83) nhưng recall thấp (0.4) — retrieval tìm đúng nhưng chưa đủ. LLM tự bịa phần còn thiếu thay vì nói không biết.
- **Suggested fix:** Tighten prompt guardrail + cải thiện chunking để tăng recall. Thêm "Chỉ trả lời những gì có trong context."

---

## Tổng hợp Pattern Failures

| Pattern | Số lần xuất hiện | Metric thấp nhất | Root cause |
|---------|-----------------|-------------------|------------|
| LLM hallucinating | 4/5 cases | faithfulness = 0.0 | Prompt không ràng buộc đủ chặt |
| Answer mismatch intent | 1/5 cases | answer_relevancy = 0.0 | Prompt template thiếu instruction restate question |
| Context recall thấp | 3/5 cases | context_recall ≤ 0.4 | Chunking chưa cover đủ ground truth |

---

## Case Study (cho presentation)

**Question chọn phân tích:** Case #3 — context_precision ≈ 1.0 nhưng faithfulness = 0.0

**Lý do chọn:** Đây là case thú vị nhất vì retrieval hoạt động gần hoàn hảo (precision ≈ 1.0) nhưng generation vẫn fail. Chứng minh rằng cải thiện retrieval (M1+M2+M3) chưa đủ — cần kiểm soát cả generation.

**Error Tree walkthrough:**
1. **Output đúng?** → **Không** — faithfulness = 0.0, LLM bịa câu trả lời.
2. **Context đúng?** → **Một phần** — context_precision ≈ 1.0 (tìm đúng document) nhưng context_recall = 0.0 (thiếu thông tin ground truth). Chunking có thể đã cắt mất phần chứa đáp án.
3. **Query rewrite OK?** → **Có thể** — query gốc đủ rõ để retrieval tìm đúng document (precision cao).
4. **Fix ở bước:**
   - **Ngắn hạn:** Thêm prompt guardrail ở generation — "Nếu context không chứa đủ bằng chứng, trả lời 'Không đủ thông tin'" → giải quyết faithfulness.
   - **Dài hạn:** Cải thiện M1 chunking (hierarchical hoặc structure-aware) để chunk cover nhiều ground truth hơn → giải quyết context_recall.

**Nếu có thêm 1 giờ, sẽ optimize:**
1. **Vá prompt generation:** Thêm instruction "Trả lời CHỈ dựa trên context. Nếu không đủ thông tin, nói rõ." + set `temperature=0` → fix faithfulness regression ngay lập tức.
2. **Vá `save_report`/pipeline:** Lưu đầy đủ `question`, `ground_truth`, `answer`, `contexts` vào failure report JSON → hỗ trợ phân tích case-level chính xác.
3. **Cache cross-encoder scores** theo `(query_hash, doc_hash)` — tiết kiệm 30-40% inference cost cho repeated queries (insight từ Thắng — M3).
4. **Bổ sung Contextual Prepend (M5):** Thêm context "chunk nằm ở section nào" trước khi embed → cải thiện recall theo style Anthropic (−49% retrieval failure).

---

## Bài học rút ra

1. **Retrieval ≠ Generation:** Cải thiện retrieval (M1+M2+M3) chỉ giải quyết nửa bài toán. Generation cần prompt engineering riêng để đảm bảo faithfulness.
2. **Fallback quan trọng:** Dependency lệch version (`underthesea`, `rank_bm25`, Qdrant API, RAGAS) là rủi ro lớn — cần fallback và pin version từ đầu.
3. **Warm-up benchmark:** Model load cost (~2GB) có thể dominate latency nếu không có warm-up — bài học từ M3 benchmark.
4. **Logging là thiết yếu:** Thiếu trường `question`/`answer`/`context` trong failure report khiến không thể truy vết lỗi cụ thể — cần fix ngay ở M4 `save_report()`.
