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
| Faithfulness | Khoảng 0.6–0.8 trong câu hỏi mở, khi câu trả lời vẫn không có claim sai nghiêm trọng. | Dưới 0.6 hoặc có claim không có evidence, đặc biệt với giá, chính sách và an toàn. | Kiểm tra grounding, prompt và nguồn context; block release nếu có hallucination nghiêm trọng. |
| Answer Relevance | Khoảng 0.6–0.8 với câu hỏi mơ hồ hoặc cần hỏi lại để làm rõ. | Dưới 0.6, câu trả lời lạc intent hoặc thường xuyên không trả lời trực tiếp. | Phân tích routing/query rewriting và bổ sung test intent; block nếu lỗi phổ biến. |
| Context Recall | Khoảng 0.6–0.8 với câu hỏi chỉ cần một phần evidence hoặc có thông tin ngoài phạm vi corpus. | Dưới 0.6 khi câu hỏi cần nhiều điều kiện nhưng retriever bỏ sót evidence bắt buộc. | Kiểm tra query, chunking và top-k; cải thiện retriever trước khi tối ưu generation. |
| Context Precision | Khoảng 0.6–0.8 khi top-k có một ít chunk nhiễu nhưng evidence đúng vẫn đứng sớm. | Dưới 0.6 khi context đầu bảng chủ yếu không liên quan, làm tăng nguy cơ trả lời sai. | Rerank/lọc chunk và điều chỉnh top-k; theo dõi precision@k cùng faithfulness. |
| Completeness | Khoảng 0.6–0.8 với câu hỏi đơn giản, expected answer có phần tùy chọn hoặc không cần liệt kê mọi chi tiết. | Dưới 0.6 hoặc bỏ sót điều kiện, ngoại lệ, bước xử lý hay cảnh báo quan trọng. | So sánh với expected answer, kiểm tra recall và prompt checklist; block nếu thiếu thông tin bắt buộc. |

### Exercise 1.2 — Bias trong LLM-as-a-Judge

Ba bias thường gặp:

- Position bias: judge ưu tiên answer xuất hiện trước.
- Verbosity bias: judge ưu tiên answer dài hơn.
- Self-preference: judge ưu tiên output giống chính model đó.

**Câu 1: Thiết kế experiment phát hiện position bias với ít nhất hai conditions.**

> *Câu trả lời:* Tạo các cặp response A/B có chất lượng tương đương và chấm hai lần: condition 1 đặt A trước B, condition 2 hoán đổi thành B trước A. Giữ nguyên question và rubric, randomize thứ tự trên nhiều case. Đo tỷ lệ chọn từng response và mức chênh lệch score giữa hai thứ tự; chênh lệch có ý nghĩa cho thấy position bias.

**Câu 2: Làm thế nào giảm verbosity bias bằng rubric design?**

> *Câu trả lời:* Chấm theo các claim đúng, evidence, độ đầy đủ và khả năng hành động; không cộng điểm chỉ vì câu trả lời dài. Quy định câu trả lời ngắn nhưng đủ ý có thể đạt điểm tối đa, đồng thời phạt lan man, lặp ý và claim ngoài phạm vi. Có thể giới hạn độ dài hợp lý và dùng ví dụ chuẩn hóa cho từng mức điểm.

**Câu 3: Tại sao cần calibrate LLM judge với human labels?**

> *Câu trả lời:* Human labels cung cấp chuẩn tham chiếu để phát hiện judge chấm lệch, không nhất quán hoặc thiên vị theo domain. Calibration giúp điều chỉnh rubric/threshold, đo agreement và xác định các edge case cần human review, nhờ đó CI gate đáng tin cậy hơn.

### Exercise 1.3 — Evaluation trong CI/CD

**Câu 1: Chọn threshold để block deployment.**

| Metric | Threshold | Lý do |
|---|---:|---|
| Faithfulness | 0.80 | Không deploy nếu câu trả lời thường xuyên chứa claim không được evidence hỗ trợ; đây là rủi ro sai thông tin trực tiếp. |
| Answer Relevance | 0.80 | Đảm bảo agent trả lời đúng intent thay vì phản hồi lạc đề hoặc né câu hỏi. |
| Completeness | 0.80 | Giữ các điều kiện, ngoại lệ và bước xử lý bắt buộc; tránh câu trả lời thiếu dẫn đến khách hàng thao tác sai. |

**Câu 2: Khi nào dùng offline evaluation, online evaluation và human review?**

> *Câu trả lời:* Dùng offline evaluation trước mỗi release, prompt/model change hoặc thay đổi retriever trên golden dataset cố định. Dùng online evaluation sau deploy để theo dõi traffic thật, drift, latency, cost và feedback liên tục. Dùng human review cho case high-stakes, khi metric vượt ngưỡng cảnh báo, case adversarial/ambiguous và để calibrate định kỳ LLM judge.

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

### Exercise 3.4 — Framework Comparison (Bonus +10)

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
