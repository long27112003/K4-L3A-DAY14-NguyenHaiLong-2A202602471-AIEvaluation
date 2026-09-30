# Day 14 — Exercises

## AI Evaluation & Benchmarking · Lab Worksheet

**Thời gian làm bài:** 14:15–17:00

**Domain:** OrbitTech Store Customer Support

Điền trực tiếp câu trả lời vào file này. Golden dataset 20 QA được viết một lần
duy nhất trong `golden_dataset.json`, không chép lại toàn bộ vào Markdown.

---

Từ 14:15–14:30, cài môi trường và chạy baseline tests theo `guide_lab.md`.

---

## Part 1 — Warm-up (14:30–14:45)

### Exercise 1.1 — RAGAS Metric Thresholds

Theo bài giảng:

- 0.8–1.0: Good — monitor, maintain.
- 0.6–0.8: Needs work — analyze failures, iterate.
- Dưới 0.6: Significant issues — investigate.

Với từng metric, xác định khi nào score thấp có thể chấp nhận và khi nào là
critical.

| Metric | Acceptable Low Score Scenario | Critical Low Score Scenario | Action Required |
|---|---|---|---|
| Faithfulness | Câu chào hỏi xã giao, câu hỏi mở ngoài tài liệu hoặc bot chủ động từ chối lịch sự ("Tôi không tìm thấy thông tin trong tài liệu"). | Trả lời sai/bịa đặt thông tin chính sách bảo hành, hoàn tiền hoặc thông số kỹ thuật sản phẩm OrbitTech (ảo giác/hallucination). | Thêm chỉ dẫn nghiêm ngặt trong prompt ("Chỉ dựa vào context, không suy diễn"), giảm temperature về 0, bổ sung cơ chế citation dẫn nguồn. |
| Answer Relevance | Khách hỏi quá ngắn/mơ hồ và bot chủ động hỏi lại để làm rõ ngữ cảnh ("Bạn muốn hỏi về mẫu laptop nào của OrbitTech?"). | Trả lời lạc đề hoàn toàn, không liên quan đến câu hỏi mua sắm hoặc hỗ trợ kỹ thuật của khách hàng. | Cải thiện System Prompt, bổ sung bước phân loại ý định (Intent Classification) trước khi sinh câu trả lời, nhắc nhở trả lời trực diện. |
| Context Recall | Câu hỏi đơn giản chỉ cần 1 ý định nghĩa ngắn gọn, không yêu cầu trích xuất toàn bộ chi tiết tài liệu nền. | Câu hỏi phức hợp (multi-hop/nhiều điều kiện) nhưng retriever bỏ sót các điều khoản quan trọng dẫn đến câu trả lời thiếu thông tin cốt lõi. | Tăng giá trị `top_k`, điều chỉnh kích thước chunk và overlap hợp lý, áp dụng Hybrid Search (BM25 + Dense vector search) hoặc Query Expansion (HyDE). |
| Context Precision | `top_k` lớn lấy thêm các chunk ngữ cảnh phụ trợ, nhưng chunk đúng vẫn nằm trong top và LLM vẫn tổng hợp chính xác. | Các chunk liên quan bị xếp ở cuối danh sách (rank thấp), chunk rác chiếm top 1-2 khiến generator bị nhiễu thông tin hoặc phân tâm. | Tích hợp thêm bước Reranking (như Cohere Rerank / Cross-Encoder) sau retrieval, fine-tune lại embedding model trên tập dữ liệu kỹ thuật của OrbitTech. |
| Completeness | Khách hàng chỉ cần câu trả lời ngắn gọn dạng Yes/No hoặc tóm tắt nhanh không cần liệt kê toàn bộ thông số. | Khách hỏi quy trình hoặc điều kiện đầy đủ (ví dụ: các bước đổi trả hàng) nhưng bot chỉ nêu 1 bước rồi dừng lại. | Cải thiện prompt yêu cầu trả lời có cấu trúc (bullet points/numbered list), tăng max_tokens, kiểm tra độ phủ các keywords/entities so với expected answer. |

### Exercise 1.2 — Bias trong LLM-as-a-Judge

Ba bias thường gặp:

- Position bias: judge ưu tiên answer xuất hiện trước.
- Verbosity bias: judge ưu tiên answer dài hơn.
- Self-preference: judge ưu tiên output giống chính model đó.

**Câu 1: Thiết kế experiment phát hiện position bias với ít nhất hai conditions.**

> *Câu trả lời:*
> Thiết kế thử nghiệm đánh giá cặp (Pairwise Evaluation) trên tập 50 câu hỏi:
> - **Condition 1 (Original Order):** Prompt LLM Judge đánh giá so sánh chất lượng giữa `[Answer A, Answer B]`.
> - **Condition 2 (Swapped Order):** Giữ nguyên câu hỏi và tiêu chí, hoán đổi vị trí hiển thị thành `[Answer B, Answer A]`.
> - **Đo lường & Kết luận:** Tính tỷ lệ bất nhất vị trí (Position Inconsistency Rate) = % số lần LLM Judge đổi lựa chọn sang đáp án khác chỉ vì vị trí của nó thay đổi (ưu tiên chọn vị trí 1 bất kể là A hay B). Nếu tỷ lệ này > 15%, mô hình có Position Bias rõ rệt. Giải pháp là chạy cả 2 chiều và chỉ ghi nhận điểm khi đồng nhất, hoặc xáo trộn ngẫu nhiên vị trí.

**Câu 2: Làm thế nào giảm verbosity bias bằng rubric design?**

> *Câu trả lời:*
> Trong Rubric chấm điểm, thiết kế rõ ràng tiêu chí **Độ cô đọng (Conciseness) & Mật độ thông tin (Fact Density)**:
> 1. Quy định rõ: "Câu trả lời dài dòng, chứa từ ngữ đệm, lặp ý hoặc không trực tiếp giải quyết câu hỏi sẽ bị trừ điểm (ví dụ: trừ 1 điểm nếu dài hơn 150 từ mà không thêm giá trị mới)."
> 2. Đưa ra hướng dẫn chấm điểm dựa trên danh sách luận điểm/sự thật cần đạt (Checklist of Atomic Facts) thay vì đánh giá cảm tính tổng thể.
> 3. Trong thang điểm 1-5, định nghĩa mức 5 là "Đầy đủ, chính xác và súc tích nhất", còn trả lời dài nhưng lan man chỉ đạt mức 3.

**Câu 3: Tại sao cần calibrate LLM judge với human labels?**

> *Câu trả lời:*
> 1. **Kiểm chứng độ tin cậy:** LLM Judge có thể mắc các thiên kiến tiềm ẩn (Leniency bias cho điểm quá rộng rãi > 0.8, hoặc Severity bias quá khắt khe < 0.3, hoặc tự ưu ái phong cách của chính họ).
> 2. **Căn chỉnh ngưỡng quyết định:** So sánh điểm của LLM Judge với tập dữ liệu con người đã gán nhãn (Human Ground-Truth) để tính hệ số tương quan (như Spearman, Pearson hoặc Cohen's Kappa). Qua đó điều chỉnh ngưỡng (threshold) hoặc hiệu chỉnh lại prompt rubric cho sát với tiêu chuẩn thực tế của doanh nghiệp.

### Exercise 1.3 — Evaluation trong CI/CD

**Câu 1: Chọn threshold để block deployment.**

| Metric | Threshold | Lý do |
|---|---:|---|
| Faithfulness | 0.85 | Ngăn ngừa ảo giác (hallucination). Trong thương mại điện tử / kỹ thuật, cung cấp sai chính sách bảo hành hoặc sai giá sẽ gây rủi ro pháp lý và thiệt hại tài chính. |
| Answer Relevance | 0.80 | Đảm bảo câu trả lời giải quyết trực tiếp thắc mắc của khách hàng, tránh trả lời vòng vo gây ức chế cho người dùng. |
| Completeness | 0.75 | Đảm bảo cung cấp đầy đủ các bước hướng dẫn hoặc điều kiện cần thiết để khách hàng có thể tự xử lý vấn đề. |

**Câu 2: Khi nào dùng offline evaluation, online evaluation và human review?**

> *Câu trả lời:*
> - **Offline Evaluation:** Dùng trong giai đoạn phát triển (Development) và cổng kiểm soát CI/CD trước khi deploy. Chạy tự động trên bộ Golden Dataset cố định để đo lường độ hồi quy (Regression Testing) nhanh chóng, chi phí thấp, an toàn tuyệt đối vì không ảnh hưởng đến người dùng thật.
> - **Online Evaluation:** Dùng khi hệ thống đã được triển khai trên môi trường Production (A/B testing, Canary deployment). Đánh giá trực tiếp trên luồng người dùng thật thông qua các tín hiệu hành vi: tỷ lệ bấm Thumbs Up/Down, tỷ lệ phiên chat hoàn thành (Task Completion Rate), thời gian phản hồi (Latency), và tỷ lệ chuyển tiếp sang tổng đài viên (Escalation Rate).
> - **Human Review:** Dùng định kỳ (đánh giá ngẫu nhiên 1-5% dữ liệu thực tế) hoặc khi xử lý các trường hợp tranh chấp/điểm số thấp bất thường. Đồng thời dùng để gán nhãn dữ liệu chuẩn giúp hiệu chuẩn (calibrate) lại LLM Judge và liên tục mở rộng bộ Golden Dataset.

---

## Part 2 — Core Coding (14:45–15:40)

Hoàn thiện các TODO bắt buộc trong `template.py`.

### Task 1 — Data Models

- `QAPair`: question, expected answer, gold context, metadata và retrieved contexts.
- `EvalResult`: answer-side scores, optional retrieval scores, pass/failure fields.
- `overall_score()`: trung bình Faithfulness, Relevance và Completeness.

### Task 2 — RAGASEvaluator

Answer-side:

- `evaluate_faithfulness(answer, context)`
- `evaluate_relevance(answer, question)`
- `evaluate_completeness(answer, expected)`

Retrieval-side:

- `evaluate_context_recall(contexts, expected)`
- `evaluate_context_precision(contexts, expected)`

Full pipeline:

- `run_full_eval(..., contexts=None)` luôn tính ba answer metrics.
- Nếu có `contexts`, tính và lưu thêm Context Recall và Context Precision.
- Retrieval scores không làm thay đổi `overall_score()` và pass rule gốc.

### Task 3 — LLMJudge

- `score_response(question, answer, rubric)`
- `detect_bias(scores_batch)`

### Task 4 — BenchmarkRunner

- `run(qa_pairs, agent_fn, evaluator)`
- `generate_report(results)`
- `run_regression(new_results, baseline_results)`
- `identify_failures(results, threshold)`

`BenchmarkRunner.run()` phải truyền `pair.retrieved_contexts` vào
`run_full_eval()`. Report phải có average của hai retrieval metrics.

### Task 5 — FailureAnalyzer

- `categorize_failures(failures)`
- `find_root_cause(failure)`
- `generate_improvement_suggestions(failures)`
- `generate_improvement_log(failures, suggestions)`

Kiểm tra:

```bash
pytest tests/ -v
```

`rerank_by_overlap()` là TODO bonus của Exercise 3.5. Test tương ứng được skip
nếu bạn chưa làm bonus.

---

## Part 3 — Golden Dataset & Real Benchmark (15:40–16:35)

### Exercise 3.1 — Build the Golden Dataset

Thiết kế và validate dataset theo Mục 5–6 trong `guide_lab.md`. Nội dung 20 QA
được điền trực tiếp trong `golden_dataset.json`; phần dưới chỉ ghi lại kết quả
và quyết định thiết kế, không chép lại toàn bộ QA.

**Kết quả dataset**

| Hạng mục | Kết quả |
|---|---|
| Tổng số records | ____ / 20 |
| Easy | ____ / 5 |
| Medium | ____ / 7 |
| Hard | ____ / 5 |
| Adversarial | ____ / 3 |
| Source documents được sử dụng | ____ / 10 |
| Validator status | PASS / FAIL |

**Ba case đại diện cho quyết định thiết kế**

| ID | Difficulty | Source document(s) | Vì sao case phù hợp với difficulty/attack type? |
|---|---|---|---|
| | | | |
| | | | |
| | | | |

**Điểm khó nhất khi xây dựng expected answer hoặc evidence là gì?**

> *Câu trả lời:*

**Xác nhận:**

- [ ] Mọi claim trong expected answer đều có evidence hỗ trợ.
- [ ] Không có questions trùng ý và không dùng kiến thức ngoài corpus.
- [ ] `python validate_golden_dataset.py` báo `PASS`.

### Exercise 3.2 — Benchmark Run

Chạy:

```bash
python domain_assistant.py
python evaluate_answers.py
```

Copy bảng terminal vào đây hoặc điền từ `artifacts/benchmark_results.json`.

| ID | Question (short) | Ctx Recall | Ctx Precision | Faithfulness | Relevance | Completeness | Overall | Passed? | Failure Type |
|---|---|---:|---:|---:|---:|---:|---:|---|---|
| E01 | | | | | | | | | |
| E02 | | | | | | | | | |
| E03 | | | | | | | | | |
| E04 | | | | | | | | | |
| E05 | | | | | | | | | |
| M01 | | | | | | | | | |
| M02 | | | | | | | | | |
| M03 | | | | | | | | | |
| M04 | | | | | | | | | |
| M05 | | | | | | | | | |
| M06 | | | | | | | | | |
| M07 | | | | | | | | | |
| H01 | | | | | | | | | |
| H02 | | | | | | | | | |
| H03 | | | | | | | | | |
| H04 | | | | | | | | | |
| H05 | | | | | | | | | |
| A01 | | | | | | | | | |
| A02 | | | | | | | | | |
| A03 | | | | | | | | | |

**Aggregate Report**

- Overall pass rate: ____%
- Avg Context Recall: ____
- Avg Context Precision: ____
- Avg Faithfulness: ____
- Avg Relevance: ____
- Avg Completeness: ____
- Failure type distribution: ____

**Ba cases có Overall Score thấp nhất**

1. ID: ____ | Score: ____ | Failure type: ____
2. ID: ____ | Score: ____ | Failure type: ____
3. ID: ____ | Score: ____ | Failure type: ____

**Nhận xét ngắn:** Metric nào yếu nhất? Kết quả gợi ý vấn đề nằm ở retrieval
hay generation?

> *Câu trả lời:*

### Exercise 3.3 — LLM-as-a-Judge Rubric Design

Thiết kế rubric domain-specific cho OrbitTech Customer Support. Mỗi mức phải
đủ cụ thể để hai người chấm độc lập có thể hiểu giống nhau.

Chọn 3–5 dimensions:

- [ ] Correctness
- [ ] Completeness
- [ ] Relevance
- [ ] Evidence/citation
- [ ] Actionability
- [ ] Safety/privacy
- [ ] Tone/clarity
- [ ] Dimension khác: __________

| Score | Tiêu chí domain-specific | Ví dụ response |
|---:|---|---|
| 5 | | |
| 4 | | |
| 3 | | |
| 2 | | |
| 1 | | |

**Ba edge cases khó chấm**

| Edge Case | Tại sao khó chấm? | Rubric xử lý thế nào? |
|---|---|---|
| | | |
| | | |
| | | |

**Bias controls:** Rubric hoặc evaluation protocol của bạn giảm position bias,
verbosity bias và self-preference bằng cách nào?

> *Câu trả lời:*

### Exercise 3.4 — Framework Comparison (Bonus +5)

Chỉ làm sau khi hoàn thành 3.1–3.3. Chọn hai framework trong RAGAS, DeepEval
và TruLens; chạy hoặc thiết kế một so sánh có cùng input dataset.

| Tiêu chí | Framework 1: ____ | Framework 2: ____ |
|---|---|---|
| Setup complexity | | |
| Metrics available | | |
| CI/CD integration | | |
| Kết quả trên cùng dataset | | |
| Insight rút ra | | |

- Scores có nhất quán không?
- Framework nào strict hơn và vì sao?
- Hai framework có tìm ra cùng failure cases không?

> *Phân tích:*

### Exercise 3.5 — Retrieval Reranking (Bonus +5)

Mục tiêu: kiểm tra việc đổi thứ tự chunks có tăng Context Precision mà không
thay đổi Context Recall hay không.

1. Chọn ít nhất 5 cases từ `artifacts/actual_answers.json`.
2. Tính Context Recall và Context Precision trước rerank.
3. Implement `rerank_by_overlap()` hoặc một reranker khác.
4. Rerank cùng tập chunks, không thêm hoặc xóa chunk.
5. Tính lại hai metrics và giải thích kết quả.

| ID | Recall before | Recall after | Precision before | Precision after | Delta Precision |
|---|---:|---:|---:|---:|---:|
| | | | | | |
| | | | | | |
| | | | | | |
| | | | | | |
| | | | | | |
| **Avg** | | | | | |

**Tại sao Recall dự kiến không đổi?**

> *Câu trả lời:*

**Khi nào reranking không đủ và cần sửa retriever/query/chunking?**

> *Câu trả lời:*

---

## Part 4 — Reflection (16:35–16:50)

Hoàn thành `reflection.md` bằng kết quả thật từ Exercise 3.2.

---

## Completion Checklist

Hoàn thành kiểm tra cuối trong khoảng 16:50–17:00.

- [ ] Tất cả required tests pass.
- [ ] `golden_dataset.json` validate thành công.
- [ ] Exercise 3.1 hoàn thành trong file JSON và bảng kết quả phía trên.
- [ ] Exercise 3.2 có năm metrics, aggregate report và ba cases thấp nhất.
- [ ] Exercise 3.3 có rubric 1–5 và bias controls.
- [ ] `reflection.md` có ba failure analyses và regression strategy.
- [ ] Đã copy `template.py` thành `solution/solution.py`.
- [ ] Exercise 3.4 và 3.5 chỉ làm nếu chọn bonus.
