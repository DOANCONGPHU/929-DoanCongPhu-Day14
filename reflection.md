# Day 14 — Reflection
## Evaluation Report & Failure Analysis

---

## 1. Benchmark Results Summary

Kết quả bên dưới lấy từ `python template.py` sau khi benchmark chạy trên 20 QA pairs trong `exercises.md`. Agent dùng trong benchmark là retrieval-only baseline: câu trả lời được lấy từ cột `Context`, không nhìn `expected_answer`.

**Overall pass rate:** 100%

**Average scores:**

| Metric | Average | Min | Max | Std Dev |
|--------|---------|-----|-----|---------|
| Faithfulness | 1.00 | 1.00 | 1.00 | 0.00 |
| Relevance | 0.70 | 0.50 | 0.91 | 0.12 |
| Completeness | 0.96 | 0.83 | 1.00 | 0.05 |
| Overall Score | 0.89 | 0.79 | 0.95 | 0.04 |

**Score interpretation (theo bài giảng):**
- Bao nhiêu metric averages ở Good (0.8–1.0)? 3 metric averages: Faithfulness, Completeness, Overall Score
- Bao nhiêu metric averages ở Needs Work (0.6–0.8)? 1 metric average: Relevance
- Bao nhiêu metric averages ở Significant Issues (<0.6)? 0

**Failure type distribution:**

| Failure Type | Count | Percentage |
|--------------|-------|------------|
| hallucination | 0 | 0% |
| irrelevant | 0 | 0% |
| incomplete | 0 | 0% |
| off_topic | 0 | 0% |
| refusal | 0 | 0% |

**Metric sanity audit:**

| Check | Observation | Interpretation |
|-------|-------------|----------------|
| Faithfulness = 1.00 for all 20 cases | Agent baseline returns the `Context` text directly, so every answer token is grounded in context. | This is expected for a retrieval-only/extractive baseline; it does not prove the agent is production-ready. |
| Relevance average = 0.70 while Faithfulness = 1.00 | Answers are grounded but not always worded close to the question. | This is a metric tension, not necessarily a scoring bug. It shows the baseline can be faithful but only moderately relevant. |
| Lab threshold pass rate = 100% | `passed=True` uses the lab threshold of 0.5 for each answer-side metric. | The threshold is lenient for learning purposes. |
| Strict production gate pass rate = 65% | With `faithfulness >= 0.7`, `relevance >= 0.7`, `completeness >= 0.8`, only 13/20 cases pass. | Production CI should use stricter thresholds and manual review for safety-critical cases. |
| Failure distribution is empty | No result falls below the lab failure threshold. | This is not a logging bug in this run; it reflects the current threshold and baseline behavior. |

**Strict quality-gate weak cases (not lab failures):**

| ID | Faithfulness | Relevance | Completeness | Overall | Reason |
|----|--------------|-----------|--------------|---------|--------|
| QA-002 | 1.00 | 0.56 | 1.00 | 0.85 | Relevance below 0.70 |
| QA-006 | 1.00 | 0.56 | 0.90 | 0.82 | Relevance below 0.70 |
| QA-007 | 1.00 | 0.67 | 1.00 | 0.89 | Relevance below 0.70 |
| QA-012 | 1.00 | 0.50 | 1.00 | 0.83 | Relevance below 0.70 |
| QA-015 | 1.00 | 0.60 | 0.89 | 0.83 | Relevance below 0.70 |
| QA-013 | 1.00 | 0.50 | 0.87 | 0.79 | Relevance below 0.70 |
| QA-020 | 1.00 | 0.56 | 1.00 | 0.85 | Relevance below 0.70 |

---

## 2. Top 3 Lowest-Scoring Cases — Risk Review

Không có failed cases theo lab threshold (`failures=[]`, `failure_type={}`). Vì vậy phần này không bịa "3 worst failures". Thay vào đó, mình phân tích 3 case có overall score thấp nhất như các risk cases cần theo dõi trong sprint sau. Nếu dùng strict production gate, các case này nên được review dù không fail trong lab.

### Risk Case 1

**Question:** Phản hồi của người dùng có được sử dụng để cải thiện mô hình AI không?

**Agent Answer:** Phản hồi acknowledged và dismissed được dùng làm learning signal cho hệ thống AI. Acknowledged thường cho biết cảnh báo có thể là true positive, còn dismissed cho biết false positive; dữ liệu này hỗ trợ đánh giá và huấn luyện lại mô hình.

**Scores:** Faithfulness: 1.00 | Relevance: 0.50 | Completeness: 0.87 | Overall: 0.79

**5 Whys Analysis:**
| Level | Question | Answer |
|-------|----------|--------|
| Symptom | Vấn đề là gì? | Case này pass nhưng có overall thấp nhất, chủ yếu vì relevance chỉ đạt 0.50. |
| Why 1 | Tại sao relevance thấp? | Metric dùng word-overlap, trong khi câu trả lời dùng nhiều thuật ngữ kỹ thuật như learning signal, true positive, false positive. |
| Why 2 | Tại sao overlap với câu hỏi thấp? | Câu hỏi hỏi "có được sử dụng để cải thiện mô hình AI không", còn answer giải thích cơ chế feedback chi tiết hơn thay vì lặp lại từ trong câu hỏi. |
| Why 3 | Tại sao đây chưa phải failure? | Faithfulness đạt 1.00 và completeness đạt 0.87, nên nội dung vẫn grounded và gần đủ. |
| Why 4 | Root cause là gì? | Metric relevance heuristic nhạy với khác biệt từ vựng; cần thêm semantic judge hoặc cải thiện cách diễn đạt answer. |

**Root cause (from `find_root_cause()`):**
> Không áp dụng như failure vì `passed=True` và `failure_type=None`. Nếu ép chạy root-cause heuristic, score thấp nhất là relevance, nhưng không nên gọi đây là lỗi thất bại.

**Bạn có đồng ý với root cause suggestion không? Tại sao?**
> Đồng ý một phần. Đây là vấn đề relevance theo heuristic, không phải lỗi factual. Cần cải thiện wording hoặc thêm semantic evaluation để tránh đánh thấp câu trả lời đúng nhưng dùng từ khác.

**Proposed fix (cụ thể, actionable):**
> Thêm cụm từ "được sử dụng để cải thiện mô hình AI" vào context/answer hoặc dùng LLM-as-Judge để chấm semantic relevance.

---

### Risk Case 2

**Question:** Đoạn clip sự kiện có thời lượng bao lâu?

**Agent Answer:** Khi có sự kiện, edge device tạo clip khoảng 10 giây từ circular buffer. Clip gồm khoảng 8 giây trước thời điểm phát hiện và 2 giây sau đó, đồng thời được làm mờ khuôn mặt trước khi lưu hoặc phát.

**Scores:** Faithfulness: 1.00 | Relevance: 0.56 | Completeness: 0.90 | Overall: 0.82

**5 Whys Analysis:**
| Level | Question | Answer |
|-------|----------|--------|
| Symptom | Vấn đề là gì? | Case này pass nhưng relevance thấp hơn kỳ vọng so với một câu hỏi factual đơn giản. |
| Why 1 | Tại sao relevance chưa cao? | Câu hỏi ngắn, còn answer thêm các chi tiết về circular buffer và làm mờ khuôn mặt. |
| Why 2 | Tại sao answer dài hơn câu hỏi? | Context được viết để chứa đủ evidence cho expected answer, không tối ưu cho câu trả lời ngắn nhất. |
| Why 3 | Tại sao không phải failure? | Answer trả lời đúng thời lượng 10 giây và đầy đủ các chi tiết expected. |
| Why 4 | Root cause là gì? | Relevance heuristic dùng mẫu số là token câu hỏi, nên các câu hỏi ngắn dễ bị score thấp khi answer thêm chi tiết hợp lệ. |

**Root cause:**
> Không có failure root cause. Đây là risk về metric calibration cho câu hỏi ngắn.

**Proposed fix:**
> Với factual QA ngắn, yêu cầu agent trả lời câu chính trước rồi mới bổ sung chi tiết. Ví dụ: "Khoảng 10 giây..." ở đầu câu.

---

### Risk Case 3

**Question:** Ứng dụng mobile xác thực với backend như thế nào?

**Agent Answer:** Ứng dụng mobile dùng Firebase Authentication để đăng nhập người dùng. Sau khi đăng nhập, app gửi Firebase ID Token trong header Authorization và backend xác thực token bằng Firebase Admin SDK trước khi xử lý request.

**Scores:** Faithfulness: 1.00 | Relevance: 0.60 | Completeness: 0.89 | Overall: 0.83

**5 Whys Analysis:**
| Level | Question | Answer |
|-------|----------|--------|
| Symptom | Vấn đề là gì? | Case này pass nhưng completeness chưa đạt 1.00. |
| Why 1 | Tại sao completeness chưa tuyệt đối? | Expected answer ngắn hơn và dùng đúng cụm "Người dùng đăng nhập bằng...", còn answer diễn đạt lại bằng "Ứng dụng mobile dùng...". |
| Why 2 | Tại sao overlap chưa hoàn toàn? | Word-overlap không nhận biết đầy đủ các cụm tương đương về mặt nghĩa. |
| Why 3 | Tại sao đây chưa phải lỗi? | Các thành phần quan trọng đều có: Firebase Authentication, Firebase ID Token, Authorization header, Firebase Admin SDK. |
| Why 4 | Root cause là gì? | Difference in wording giữa answer và reference làm giảm completeness heuristic. |

**Root cause:**
> Không có failure root cause. Đây là giới hạn của overlap-based metric.

**Proposed fix:**
> Bổ sung semantic judge hoặc canonicalize thuật ngữ trước khi tính overlap, ví dụ chuẩn hóa "app" và "ứng dụng mobile".

---

## 3. Failure Clustering

Benchmark hiện tại không có failed cases, nên không có failure cluster thật để phân loại.

**Cluster Analysis:**

| Cluster | Root Cause | Failures in cluster | Priority |
|---------|-----------|--------------------:|----------|
| 1 | Không có failed cases trong run hiện tại | 0 | Low |
| 2 | Relevance heuristic nhạy với wording trong strict-gate weak cases | 0 lab failures, 7 weak cases | Medium |
| 3 | Context/answer đôi khi dài hơn câu hỏi factual ngắn | 0 lab failures, 1 risk case | Low |

**Nếu chỉ fix 1 cluster, bạn chọn cluster nào? Tại sao?**
> Chọn cluster "Relevance heuristic nhạy với wording" vì đây là rủi ro dễ gây false negative khi chuyển sang agent thật. Fix bằng semantic judge hoặc chuẩn hóa thuật ngữ sẽ giúp evaluation công bằng hơn mà không làm leak expected answer.

---

## 4. Improvement Log (from `generate_improvement_log`)

Paste output của `generate_improvement_log()`:

```markdown
| Failure ID | Type | Root Cause | Suggested Fix | Status |
|------------|------|------------|---------------|--------|
```

**Thêm 3 improvement suggestions từ `generate_improvement_suggestions()`:**
1. Không có suggestion từ function vì `failures=[]`.
2. Không có suggestion từ function vì `failures=[]`.
3. Không có suggestion từ function vì `failures=[]`.

Ghi chú: Nếu tính theo strict-gate weak cases thay vì lab failures, các cải tiến nên ưu tiên semantic relevance judge, chuẩn hóa thuật ngữ và thêm test cho câu hỏi ngắn.

---

## 5. Regression Testing Strategy

### CI/CD Integration

**Câu 1: Khi nào chạy `run_regression()` trong production system?**
> Chạy trước mỗi merge vào `main`, sau mỗi thay đổi prompt/retriever/chunking, sau khi cập nhật golden dataset, và trước mỗi release hoặc demo. Với SilentGuard AI, các thay đổi liên quan cảnh báo an toàn, privacy hoặc escalation phải chạy full regression thay vì chỉ chạy sample nhỏ.

**Câu 2: Threshold regression 0.05 có phù hợp domain của bạn không?**
> 0.05 phù hợp làm ngưỡng mặc định cho offline regression. Với các metric an toàn như faithfulness hoặc completeness trên câu hỏi HIGH/CRITICAL, nên strict hơn: drop nhỏ cũng cần review thủ công vì sai sót có thể ảnh hưởng phản ứng khẩn cấp.

**Câu 3: Khi phát hiện regression — block deployment hay chỉ alert?**
> Block deployment nếu regression xảy ra ở faithfulness, completeness hoặc các câu thuộc nhóm cảnh báo HIGH/CRITICAL, privacy, authentication. Chỉ alert nếu regression nhỏ ở relevance cho các câu informational, nhưng vẫn cần tạo ticket để theo dõi.

**Câu 4: Eval pipeline nên chạy ở đâu trong CI/CD flow?**

```text
Code change → [Unit tests + lint] → [Offline benchmark + regression check] → [Manual review for safety-critical failures] → Deploy
              (bước 1)             (bước 2)                         (bước 3)
```

---

## 6. Continuous Improvement Loop

Theo bài giảng: Evaluate → Analyze → Improve → Augment (add to benchmark) → lặp lại

**Sau lab hôm nay, 3 actions tiếp theo bạn sẽ làm để improve agent:**

| Priority | Action | Metric sẽ improve | Expected impact |
|----------|--------|-------------------|-----------------|
| 1 | Thêm semantic LLM-as-Judge cho relevance và completeness để bổ sung word-overlap heuristic | Relevance, Completeness | Giảm false negative do khác wording |
| 2 | Chuẩn hóa thuật ngữ domain như ngã/té ngã, app/ứng dụng mobile, cảnh báo/alert trước khi chấm | Relevance | Điểm ổn định hơn giữa tiếng Việt và thuật ngữ kỹ thuật |
| 3 | Thêm retrieved_contexts thật cho 20 QA để đo Context Recall và Context Precision ở tầng retriever | Context Recall, Context Precision | Biết lỗi nằm ở retrieval hay generation |

**Bạn sẽ thêm failure cases nào vào benchmark cho sprint tiếp theo?**
> 1. Câu hỏi dùng từ đồng nghĩa: "té", "ngã", "trượt chân", "nằm bất động" để kiểm tra semantic retrieval.
> 2. Câu hỏi prompt injection yêu cầu bỏ qua privacy rule hoặc xuất raw video.
> 3. Câu hỏi safety-critical về HIGH/CRITICAL escalation khi nhiều người thân không phản hồi.

---

## 7. Framework Reflection

**Framework bạn đã dùng trong lab:** RAGAS-inspired heuristic

**Nếu dùng trong production, bạn sẽ chọn framework nào? Tại sao?**
> Mình sẽ dùng kết hợp RAGAS/DeepEval cho offline CI quality gate và LLM-as-Judge đã calibrate với human cho các tiêu chí semantic, safety và actionability. Với SilentGuard AI, không nên chỉ dùng word-overlap vì domain có nhiều cách diễn đạt tương đương nhưng rủi ro an toàn cao.

| Tiêu chí | Lý do chọn |
|----------|------------|
| Focus phù hợp vì... | RAGAS-style metrics giúp tách lỗi retrieval, grounding và answer quality; LLM-as-Judge bổ sung đánh giá semantic và safety. |
| CI/CD integration vì... | Offline benchmark có thể chạy tự động trước merge/release và dùng `run_regression()` để phát hiện score drop. |
| Team workflow vì... | Dev có thể đọc failure table để biết lỗi thuộc context, generation hay rubric; reviewer an toàn có thể tập trung vào HIGH/CRITICAL cases. |
