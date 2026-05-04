# Failure Analysis — Lab 18: Production RAG

**Nhóm:** Production RAG Group  
**Thành viên:** → M1 · Lê Quý Công → M2 · Nguyễn Ngọc Thắng → M3 · Trần Minh Toàn → M4 · 

---

## RAGAS Scores

| Metric | Naive Baseline | Production | Δ |
|--------|---------------|------------|---|
| Faithfulness | 0.8224 | 0.6375 | -0.1849 |
| Answer Relevancy | 0.3321 | 0.3419 | +0.0098 |
| Context Precision | 0.4292 | 0.8000 | +0.3708 |
| Context Recall | 0.3500 | 0.6700 | +0.3200 |

> Scores được tính tự động bởi `naive_baseline.py` và `main.py` → `reports/naive_baseline_report.json`, `reports/ragas_report.json`.

---

## Bottom-5 Failures

`reports/ragas_report.json` hiện lưu 10 failure entries. Trong đó các field `question`, `ground_truth`, `answer` đang rỗng (`""`), nhưng các trường metric/diagnosis vẫn có giá trị. Dưới đây là 5 case đầu tiên theo đúng dữ liệu report.

### #1
- **Question:** _(missing in report)_
- **Expected:** _(missing in report)_
- **Got:** _(missing in report)_
- **Worst metric:** faithfulness = 0.0
- **Scores:** faithfulness=0.0, answer_relevancy=0.0, context_precision=0.0, context_recall=0.0
- **Diagnosis (report):** LLM hallucinating
- **Suggested fix (report):** Tighten prompt: 'Trả lời CHỈ dựa trên context'. Lower temperature to 0.

### #2
- **Question:** _(missing in report)_
- **Expected:** _(missing in report)_
- **Got:** _(missing in report)_
- **Worst metric:** faithfulness = 0.0
- **Scores:** faithfulness=0.0, answer_relevancy=0.0, context_precision=0.0, context_recall=0.0
- **Diagnosis (report):** LLM hallucinating
- **Suggested fix (report):** Tighten prompt: 'Trả lời CHỈ dựa trên context'. Lower temperature to 0.

### #3
- **Question:** _(missing in report)_
- **Expected:** _(missing in report)_
- **Got:** _(missing in report)_
- **Worst metric:** faithfulness = 0.0
- **Scores:** faithfulness=0.0, answer_relevancy=0.0, context_precision=0.9999999999, context_recall=0.0
- **Diagnosis (report):** LLM hallucinating
- **Suggested fix (report):** Tighten prompt: 'Trả lời CHỈ dựa trên context'. Lower temperature to 0.

### #4
- **Question:** _(missing in report)_
- **Expected:** _(missing in report)_
- **Got:** _(missing in report)_
- **Worst metric:** answer_relevancy = 0.0, faithfulness = 0.5
- **Scores:** faithfulness=0.5, answer_relevancy=0.0, context_precision=0.0, context_recall=0.5
- **Diagnosis (report):** Answer doesn't match question intent
- **Suggested fix (report):** Improve prompt template; ensure question is restated in the answer.

### #5
- **Question:** _(missing in report)_
- **Expected:** _(missing in report)_
- **Got:** _(missing in report)_
- **Worst metric:** faithfulness = 0.0
- **Scores:** faithfulness=0.0, answer_relevancy=0.0, context_precision=0.8333333332916666, context_recall=0.4
- **Diagnosis (report):** LLM hallucinating
- **Suggested fix (report):** Tighten prompt: 'Trả lời CHỈ dựa trên context'. Lower temperature to 0.

---

## Case Study (cho presentation)

**Question chọn phân tích:**  
*Một mẫu failure có `faithfulness=0.0` trong production report (question text bị thiếu trong file JSON).* 

**Error Tree walkthrough:**
1. **Output đúng?** → **Không** — faithfulness chạm 0.0.
2. **Context đúng?** → **Chưa chắc** — report không lưu text question/answer/context nên chưa truy ngược trực tiếp được.
3. **Query rewrite OK?** → **Chưa đánh giá được từ artifact hiện tại**.
4. **Fix ở bước:** M4 logging (lưu đầy đủ question/answer/context trong report) + prompt guardrail để giảm hallucination.

**Nếu có thêm 1 giờ, sẽ optimize:**
- Vá `save_report`/pipeline để không làm rỗng trường question, ground_truth, answer trong `failures`.
- Bổ sung debug dump cho top-k contexts mỗi câu hỏi để truy vết lỗi chuẩn xác theo từng case.
- Tăng ràng buộc generation: nếu context thiếu bằng chứng thì trả lời "Không đủ thông tin trong ngữ cảnh".
