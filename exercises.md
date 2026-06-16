# Day 14 — Exercises
## AI Evaluation & Benchmarking | Lab Worksheet

**Lab Duration:** 3 hours

---

## Part 1 — Warm-up (0:00–0:20)

### Exercise 1.1 — RAGAS Metric Thresholds

Theo bài giảng, score interpretation:
- 0.8–1.0: Good (Monitor, maintain)
- 0.6–0.8: Needs work (Analyze failures, iterate)
- < 0.6: Significant issues (Deep investigation)

Cho mỗi RAGAS metric, xác định khi nào score thấp là acceptable vs critical:

| Metric | Acceptable Low Score Scenario | Critical Low Score Scenario | Action Required |
|---------|-------------------------------|-----------------------------|----------------|
| **Faithfulness (Tính trung thực)** | Các xác nhận (claims) phụ, không quan trọng thiếu dẫn chứng nhưng thông tin cốt lõi vẫn đúng | Bịa đặt thông tin (hallucination), sai lệch chính sách hoặc dữ liệu quan trọng (Ví dụ: chatbot Air Canada bịa chính sách hoàn tiền) | Kiểm tra lại prompt grounding, ép buộc trích dẫn (citation-grounding), hoặc tăng cường guardrail về trung thực |
| **Answer Relevancy (Sự phù hợp)** | Câu trả lời quá dài dòng hoặc bao gồm thêm các thông tin bên lề không trực tiếp liên quan nhưng vẫn giải quyết được câu hỏi | Trả lời lạc đề, chung chung hoặc hoàn toàn không đáp ứng được ý định của người dùng | Tinh chỉnh prompt để bám sát câu hỏi hơn, hoặc kiểm tra xem truy vấn của người dùng có quá mơ hồ không |
| **Context Recall (Khả năng thu hồi)** | Thiếu một vài chi tiết bổ trợ nhỏ trong reference answer, nhưng bằng chứng chính vẫn có mặt trong context | Thông tin cốt lõi cần thiết để trả lời câu hỏi hoàn toàn không xuất hiện trong các đoạn văn bản được truy xuất (retrieval miss) | Sửa retriever trước tiên: Cải thiện quy trình Ingestion (re-index), thay đổi chiến lược chunking hoặc nâng cấp thuật toán tìm kiếm |
| **Context Precision (Độ chính xác ngữ cảnh)** | Thông tin đúng có xuất hiện trong top-k nhưng không nằm ở vị trí đầu tiên (vẫn đủ để generator đọc được) | Nhiễu (noise) quá nhiều khiến thông tin đúng bị đẩy xuống quá sâu, dẫn đến việc generator bỏ qua hoặc chọn sai thông tin | Triển khai hoặc tinh chỉnh bước Re-ranking để đưa các đoạn văn bản liên quan nhất lên đầu danh sách |
| **Completeness (Tính đầy đủ)** | Câu trả lời ngắn gọn, súc tích, chỉ cung cấp đúng những gì được hỏi mà không có các chi tiết mở rộng không bắt buộc | Thiếu các thành phần thiết yếu của câu trả lời (ví dụ: chỉ trả lời được 1 trong 2 ý của câu hỏi phức hợp) | Điều chỉnh system prompt để yêu cầu câu trả lời toàn diện hơn, hoặc kiểm tra xem Context Recall có đang là nút thắt cổ chai không |


---

### Exercise 1.2 — Position Bias in LLM-as-Judge

Từ bài giảng, 3 loại bias trong LLM-as-Judge:
- **Position Bias:** Judge ưu tiên answer xuất hiện trước
- **Verbosity Bias:** Judge cho điểm cao hơn answer dài hơn
- **Self-Preference:** GPT-4 judge ưu tiên GPT-4 output

**Câu 1: Thiết kế experiment phát hiện Position Bias**
> *Mô tả thí nghiệm với ít nhất 2 conditions:* 

>Condition 1 (Original): Đưa cho Judge câu hỏi kèm theo hai câu trả lời theo thứ tự: Answer A xuất hiện trước, Answer B xuất hiện sau. Ghi lại kết quả Judge chọn câu nào tốt hơn.

>Condition 2 (Swapped): Vẫn dùng câu hỏi và hai câu trả lời đó, nhưng đảo ngược thứ tự: Answer B xuất hiện trước, Answer A xuất hiện sau

**Câu 2: Làm sao fix Verbosity Bias trong rubric design?**
> *Để khắc phục Verbosity Bias (thiên kiến độ dài) ngay từ khâu thiết kế Rubric, bạn cần thực hiện các kỹ thuật sau:*
```
Thêm chỉ thị trực tiếp vào Prompt: Trong phần hướng dẫn chấm điểm, phải ghi rõ câu lệnh: "Length is NOT a factor; concise is fine" (Độ dài không phải là một yếu tố; trả lời súc tích vẫn tốt).

Thiết kế Rubric đa chiều (Multi-dimension): Thay vì chấm điểm tổng thể dễ bị đánh lừa bởi độ dài, hãy chia nhỏ tiêu chí chấm điểm thành các trục riêng biệt như tính trung thực (faithfulness), tính chính xác (correctness) và yêu cầu Judge chỉ tập trung vào các tiêu chí này.

Bắt buộc giải thích (Force rationale): Yêu cầu Judge phải thực hiện bước suy luận (Reasoning) từng bước trước khi đưa ra điểm số để tránh việc Judge chấm điểm cao một cách hời hợt chỉ vì thấy câu trả lời dài
```
**Câu 3: Tại sao cần "calibrate against human" theo best practices?**
> *Việc hiệu chỉnh so với con người (calibrate against human) là bắt buộc vì:*
```
Đảm bảo độ tin cậy: Một Judge chưa được hiệu chỉnh so với chuyên gia thì không được coi là bằng chứng (evidence), mà chỉ là một "ý kiến đắt tiền".

Đo lường sự đồng thuận: Bạn cần báo cáo chỉ số đồng thuận đã hiệu chỉnh ngẫu nhiên (chance-corrected agreement), ví dụ như hệ số Cohen’s κ, để biết Judge LLM bám sát các chuyên gia (SME) đến mức nào.

Phát hiện sai lệch theo phân đoạn (Slice analysis): Sự đồng thuận tổng thể có thể rất cao, nhưng việc đối chiếu với người giúp phát hiện các "slice" (tình huống cụ thể, độ khó cụ thể) mà Judge LLM thường xuyên chấm sai nặng.

Xác định "trần" chất lượng: Trong khi sự đồng thuận giữa người với người có thể đạt κ≈0.97, thì Judge tốt nhất (như GPT-4) hiện nay chỉ đạt khoảng κ≈0.84. Hiệu chỉnh giúp ta hiểu được khoảng cách và giới hạn của Judge tự động.
```
---

### Exercise 1.3 — Evaluation trong CI/CD

Theo bài giảng: "Agent không pass eval = không được deploy, giống unit test."

**Câu 1: Bạn sẽ set threshold nào cho từng metric trong CI/CD pipeline?**

| Metric | Threshold (block deploy nếu dưới) | Lý do |
|--------|-----------------------------------|-------|
| **Faithfulness** | 0.7 – 0.8 | Thấp hơn ngưỡng này đồng nghĩa với việc AI đang bịa đặt thông tin (hallucination). Đây là lỗi nghiêm trọng có thể dẫn đến hậu quả pháp lý như vụ Air Canada bị kiện vì chatbot bịa chính sách. |
| **Answer Relevancy** | 0.8 | Đảm bảo câu trả lời bám sát ý định người dùng. Điểm thấp cho thấy bot đang trả lời lạc đề hoặc quá chung chung, làm giảm trải nghiệm người dùng. |
| **Completeness (Agent Goal Accuracy)** | 0.8 | Đối với agent, đây là thước đo liệu mục tiêu của người dùng có thực sự được hoàn thành hay không (Task Success Rate). Đây là giá trị cốt lõi (North Star metric) của một agent. |


**Câu 2: Khi nào nên chạy offline eval vs online eval?**
>Nên chạy Offline Eval (Full suite hoặc Targeted) khi:
```
Mỗi khi có code release: Để thực hiện Regression check (kiểm tra xem code mới có làm hỏng tính năng cũ không)

Mỗi khi thay đổi Prompt: Để đảm bảo thay đổi nhỏ không gây ra tác động tiêu cực dây chuyền

Trước khi Demo hoặc Launch: Để tạo sự tự tin về chất lượng hệ thống trước các bên liên quan (stakeholders)

Đặc điểm: Chạy trên bộ dữ liệu cố định (Golden Dataset) trong môi trường Dev/CI
```
>Nên chạy Online Eval (Continuous monitoring) khi:
```
Hệ thống đã lên Production: Theo dõi liên tục trên live traffic (dữ liệu thật từ người dùng)

Mục tiêu: Phát hiện sớm tình trạng sụt giảm chất lượng (degradation) mà môi trường offline không bắt được

Kết hợp Guardrails: Có thể chạy inline để chặn các phản hồi xấu (về an toàn, độc hại) trước khi trả về cho user

```
---

## Part 2 — Core Coding (0:20–1:20)

Implement all TODOs in `template.py`. Focus on:

### Task 1: Data Models
- `QAPair` dataclass: question, expected_answer, context, metadata
- `EvalResult` dataclass: qa_pair, actual_answer, faithfulness, relevance, completeness, passed, failure_type
- `overall_score()` method: average of 3 metrics

### Task 2: RAGASEvaluator (answer-side)
- `evaluate_faithfulness(answer, context)` → word overlap heuristic
- `evaluate_relevance(answer, question)` → word overlap heuristic  
- `evaluate_completeness(answer, expected)` → word overlap heuristic
- `run_full_eval(...)` → combine all 3 + determine failure_type

### Task 2b: RAGASEvaluator (retrieval-side — chấm bước get context)
- `evaluate_context_recall(contexts, expected)` → union coverage của expected
- `evaluate_context_precision(contexts, expected)` → rank-aware Average Precision
- `rerank_by_overlap(contexts, query)` → reranker lexical (dùng ở Exercise 3.5)

### Task 3: LLMJudge
- `score_response(question, answer, rubric)` → build prompt, call judge, parse scores
- `detect_bias(scores_batch)` → check positional, leniency, severity bias

### Task 4: BenchmarkRunner
- `run(qa_pairs, agent_fn, evaluator)` → run all pairs through agent + eval
- `generate_report(results)` → aggregate stats
- `run_regression(new_results, baseline_results)` → detect drops > 0.05
- `identify_failures(results, threshold)` → filter below threshold

### Task 5: FailureAnalyzer
- `categorize_failures(failures)` → group by type
- `find_root_cause(failure)` → suggest cause based on lowest score
- `generate_improvement_suggestions(failures)` → prioritized fix list
- `generate_improvement_log(failures, suggestions)` → Markdown table output

**Verify:** `pytest tests/ -v`

---

## Part 3 — Extended Exercises (1:20–2:20)

### Exercise 3.1 — Build Your Golden Dataset (Stratified Sampling)

Theo bài giảng, golden dataset cần:
- Expert-written expected answers
- Stratified sampling theo difficulty
- Cover tất cả use cases chính
- Có edge cases và adversarial inputs

**Tạo 20 QA pairs cho domain của bạn (từ Day 2):**

#### Easy (5 pairs) — Factual lookup, single-doc
| ID | Question | Expected Answer | Context (1–2 sentences) | Source Doc |
| --- | --- | --- | --- | --- |
| QA-001 | SilentGuard AI là gì? | SilentGuard AI là hệ thống giám sát an toàn thụ động cho người cao tuổi sống một mình, dùng camera và AI trên edge để phát hiện té ngã và gửi cảnh báo. | SilentGuard AI là một hệ thống giám sát an toàn thụ động cho người cao tuổi sống một mình. Hệ thống dùng camera và AI chạy trên edge device để phát hiện tình huống té ngã, sau đó gửi cảnh báo tới gia đình hoặc người chăm sóc. | Project Docs |
| QA-002 | Đối tượng sử dụng chính của SilentGuard AI là ai? | Người cao tuổi sống một mình; ứng dụng dùng bởi thành viên gia đình hoặc người chăm sóc; quản trị viên quản lý camera và ngưỡng cảnh báo. | Đối tượng được giám sát chính là người cao tuổi sống một mình. Ứng dụng được dùng bởi thành viên gia đình hoặc người chăm sóc, còn quản trị viên phụ trách quản lý camera, hộ gia đình và ngưỡng cảnh báo. | Project Docs |
| QA-006 | Đoạn clip sự kiện có thời lượng bao lâu? | Khoảng 10 giây gồm ~8 giây trước phát hiện và ~2 giây sau, lấy từ circular buffer và đã làm mờ khuôn mặt. | Khi có sự kiện, edge device tạo clip khoảng 10 giây từ circular buffer. Clip gồm khoảng 8 giây trước thời điểm phát hiện và 2 giây sau đó, đồng thời được làm mờ khuôn mặt trước khi lưu hoặc phát. | Technical Plan |
| QA-007 | SilentGuard AI có những mức độ cảnh báo nào? | Bốn mức: LOW, MEDIUM, HIGH, CRITICAL, xác định dựa trên thời gian bất động và ngưỡng cấu hình. | SilentGuard AI phân loại cảnh báo thành bốn mức: LOW, MEDIUM, HIGH và CRITICAL. Mức độ được xác định dựa trên thời gian bất động, tín hiệu té ngã và các ngưỡng cấu hình của hộ gia đình. | Project Docs |
| QA-008 | Khi nào một sự kiện được phân loại là LOW? | Theo mặc định, bất động dưới 30 giây được phân loại LOW và thường chỉ ghi lịch sử, không gửi push notification. | Theo cấu hình mặc định, sự kiện có thời gian bất động dưới 30 giây được phân loại là LOW. Các cảnh báo LOW thường chỉ được ghi vào lịch sử để theo dõi và không gửi push notification nhằm tránh làm phiền gia đình. | Settings Docs |

#### Medium (7 pairs) — Multi-step reasoning, 2–3 docs
| ID | Question | Expected Answer | Context (1–2 sentences) | Source Doc |
| --- | --- | --- | --- | --- |
| QA-009 | Khi nào hệ thống gửi push notification? | Hệ thống gửi push khi sự kiện được phân loại từ MEDIUM trở lên; LOW thường chỉ logged_only. | Backend chỉ gửi push notification khi sự kiện được phân loại từ MEDIUM trở lên. Với mức LOW, hệ thống thường đặt trạng thái logged_only và chỉ lưu lịch sử thay vì đẩy thông báo tức thời. | Backend Docs |
| QA-011 | Gia đình có thể làm gì khi nhận được cảnh báo? | Mở app, xem thông tin sự kiện và clip đã làm mờ, chọn acknowledged để xác nhận hoặc dismissed nếu sai. | Khi nhận cảnh báo, thành viên gia đình mở ứng dụng để xem thông tin sự kiện, thời gian, mức độ cảnh báo và clip đã làm mờ. Người dùng có thể chọn acknowledged nếu đã nhận và xử lý, hoặc dismissed nếu xác định cảnh báo sai. | App Docs |
| QA-012 | Nút acknowledged và dismissed có ý nghĩa gì? | Acknowledged nghĩa đã nhận và xử lý; Dismissed nghĩa xác định cảnh báo không phải té ngã; phản hồi lưu trong alert_reviews. | Trạng thái acknowledged nghĩa là người dùng đã nhận cảnh báo và đang xử lý sự kiện. Trạng thái dismissed nghĩa là người dùng xác định cảnh báo không phải té ngã; cả hai loại phản hồi được lưu trong bảng alert_reviews để phân tích sau. | Backend Docs |
| QA-014 | Tại sao SilentGuard AI sử dụng edge device? | Edge device xử lý gần camera để giảm độ trễ, giảm phụ thuộc internet và bảo vệ quyền riêng tư vì raw video không rời nhà. | SilentGuard AI sử dụng edge device để xử lý video gần camera, nhờ đó giảm độ trễ phát hiện té ngã và giảm phụ thuộc vào kết nối internet. Cách này cũng bảo vệ quyền riêng tư vì raw video không rời khỏi nhà người dùng. | Technical Plan |
| QA-015 | Ứng dụng mobile xác thực với backend như thế nào? | Người dùng đăng nhập bằng Firebase Authentication và gửi Firebase ID Token trong header Authorization; backend kiểm tra bằng Firebase Admin SDK. | Ứng dụng mobile dùng Firebase Authentication để đăng nhập người dùng. Sau khi đăng nhập, app gửi Firebase ID Token trong header Authorization và backend xác thực token bằng Firebase Admin SDK trước khi xử lý request. | Backend Docs |
| QA-017 | API nào được dùng để lấy danh sách cảnh báo? | GET /api/alerts với tham số status, limit, offset và household_id để lọc và phân trang. | Danh sách cảnh báo được lấy qua endpoint GET /api/alerts. Endpoint này hỗ trợ các tham số status, limit, offset và household_id để lọc theo trạng thái, giới hạn số bản ghi, phân trang và lọc theo hộ gia đình. | API Docs |
| QA-018 | API nào được dùng để xem chi tiết một sự kiện và phát clip? | GET /api/events/{event_id} trả về chi tiết sự kiện và clip_url là signed URL tạm thời để phát clip đã làm mờ từ Supabase Storage. | Chi tiết sự kiện được lấy bằng GET /api/events/{event_id}. Phản hồi bao gồm thông tin sự kiện và clip_url, là signed URL tạm thời dùng để phát clip đã làm mờ được lưu trong Supabase Storage. | API Docs |

#### Hard (5 pairs) — Complex/ambiguous, nhiều cách hiểu
| ID | Question | Expected Answer | Context (1–2 sentences) | Source Doc |
| --- | --- | --- | --- | --- |
| QA-003 | SilentGuard AI phát hiện người cao tuổi bị ngã như thế nào? | Camera gửi video tới edge device; YOLOv8-Pose phát hiện keypoints; bộ phân loại té ngã phân tích góc thân, tốc độ, tỷ lệ bounding box và thời gian bất động để quyết định. | SilentGuard AI phát hiện người cao tuổi bị ngã bằng cách gửi luồng video từ camera tới edge device để xử lý cục bộ. YOLOv8-Pose phát hiện keypoints cơ thể, sau đó bộ phân loại té ngã phân tích góc thân, tốc độ chuyển động, tỷ lệ bounding box và thời gian bất động trước khi quyết định có tạo cảnh báo hay không. | Technical Plan |
| QA-004 | Video từ camera có được gửi toàn bộ lên cloud không? | Không; raw video chỉ xử lý trong RAM của edge device và không lưu lên cloud; cloud chỉ nhận metadata và clip đã làm mờ. | Raw video từ camera không được gửi toàn bộ lên cloud. Video thô chỉ được xử lý trong RAM của edge device, còn cloud chỉ nhận metadata sự kiện và clip ngắn đã làm mờ khuôn mặt. | Backend Design |
| QA-005 | SilentGuard AI bảo vệ quyền riêng tư của người cao tuổi như thế nào? | Xử lý tại edge, không upload raw video, làm mờ khuôn mặt trước khi tạo clip, lưu clip ẩn danh trong Supabase Storage và phát qua signed URL có thời hạn. | SilentGuard AI bảo vệ quyền riêng tư của người cao tuổi bằng cách xử lý video tại edge và không upload raw video. Trước khi tạo clip sự kiện, hệ thống làm mờ khuôn mặt, lưu clip ẩn danh trong Supabase Storage và chỉ phát qua signed URL có thời hạn. | Backend Design |
| QA-013 | Phản hồi của người dùng có được sử dụng để cải thiện mô hình AI không? | Có; acknowledged và dismissed dùng làm learning signal để xác định true positive và false positive, hỗ trợ đánh giá và huấn luyện lại mô hình. | Phản hồi acknowledged và dismissed được dùng làm learning signal cho hệ thống AI. Acknowledged thường cho biết cảnh báo có thể là true positive, còn dismissed cho biết false positive; dữ liệu này hỗ trợ đánh giá và huấn luyện lại mô hình. | AI Docs |
| QA-019 | Người dùng có thể thay đổi ngưỡng cảnh báo không? | Có; GET /api/settings/thresholds để lấy cấu hình và PUT /api/settings/thresholds để cập nhật ngưỡng LOW, MEDIUM, HIGH, khoảng chống cảnh báo trùng và khung giờ giảm cảnh báo. | Người dùng hoặc quản trị viên hộ gia đình có thể tùy chỉnh ngưỡng cảnh báo. API GET /api/settings/thresholds dùng để lấy cấu hình, còn PUT /api/settings/thresholds dùng để cập nhật ngưỡng LOW, MEDIUM, HIGH, khoảng chống cảnh báo trùng và khung giờ giảm cảnh báo. | Settings Docs |

#### Adversarial (3 pairs) — Out-of-scope, cố tình phá
| ID | Question | Expected Answer | Context (1–2 sentences) | Source Doc |
| --- | --- | --- | --- | --- |
| QA-010 | Điều gì xảy ra khi có cảnh báo HIGH hoặc CRITICAL nhưng gia đình không phản hồi? | Backend đặt thời điểm escalation; nếu không có phản hồi sau thời gian quy định, hệ thống gọi hoặc gửi thông báo tới liên hệ khẩn cấp tiếp theo; CRITICAL gửi tới nhiều liên hệ hơn. | Với cảnh báo HIGH hoặc CRITICAL, backend đặt thời điểm escalation sau khi gửi cảnh báo đầu tiên. Nếu gia đình không phản hồi trong thời gian quy định, hệ thống gọi hoặc gửi thông báo tới liên hệ khẩn cấp tiếp theo; riêng CRITICAL có thể gửi tới nhiều liên hệ hơn. | Technical Plan |
| QA-016 | Edge device có sử dụng Firebase Token để gọi backend không? | Không; mỗi camera/edge device dùng device API key riêng gửi qua header X-Device-Key và backend lưu hash của key để xác thực thay vì lưu key gốc. | Edge device không dùng Firebase Token của người dùng để gọi backend. Mỗi camera hoặc edge device dùng một device API key riêng trong header X-Device-Key; backend chỉ lưu hash của key để xác thực và giảm rủi ro lộ khóa gốc. | Backend Docs |
| QA-020 | Điều gì xảy ra khi camera mất kết nối? | Edge device gửi heartbeat định kỳ; nếu không nhận heartbeat trong khoảng thời gian quy định (ví dụ >5 phút), backend tạo sự kiện camera offline và gửi thông báo tới gia đình. | Khi camera mất kết nối, edge device không gửi được heartbeat định kỳ để báo camera vẫn hoạt động. Nếu backend không nhận heartbeat trong khoảng thời gian quy định, ví dụ quá 5 phút, hệ thống tạo sự kiện camera offline và gửi thông báo tới gia đình. | Backend Docs |

---

### Exercise 3.2 — Benchmark Run

Chạy `BenchmarkRunner` trên 20 QA pairs. Ghi lại kết quả:

| ID | Question (short) | Faithfulness | Relevance | Completeness | Overall | Passed? | Failure Type |
|----|-----------------|--------------|-----------|--------------|---------|---------|--------------|
| QA-001 | SilentGuard AI là gì? | 1.00 | 0.75 | 1.00 | 0.92 | Yes | None |
| QA-002 | Đối tượng sử dụng chính của SilentGuard AI | 1.00 | 0.56 | 1.00 | 0.85 | Yes | None |
| QA-006 | Đoạn clip sự kiện có thời lượng bao lâu? | 1.00 | 0.56 | 0.90 | 0.82 | Yes | None |
| QA-007 | Các mức độ cảnh báo | 1.00 | 0.67 | 1.00 | 0.89 | Yes | None |
| QA-008 | Khi nào sự kiện là LOW? | 1.00 | 0.70 | 1.00 | 0.90 | Yes | None |
| QA-009 | Khi nào gửi push notification? | 1.00 | 0.86 | 1.00 | 0.95 | Yes | None |
| QA-011 | Gia đình làm gì khi nhận cảnh báo? | 1.00 | 0.82 | 0.95 | 0.92 | Yes | None |
| QA-012 | Acknowledged và dismissed | 1.00 | 0.50 | 1.00 | 0.83 | Yes | None |
| QA-014 | Tại sao dùng edge device? | 1.00 | 0.75 | 1.00 | 0.92 | Yes | None |
| QA-015 | Mobile xác thực backend thế nào? | 1.00 | 0.60 | 0.89 | 0.83 | Yes | None |
| QA-017 | API lấy danh sách cảnh báo | 1.00 | 0.80 | 0.93 | 0.91 | Yes | None |
| QA-018 | API xem chi tiết sự kiện và clip | 1.00 | 0.79 | 0.88 | 0.89 | Yes | None |
| QA-003 | Phát hiện người cao tuổi bị ngã | 1.00 | 0.75 | 1.00 | 0.92 | Yes | None |
| QA-004 | Video có gửi toàn bộ lên cloud không? | 1.00 | 0.91 | 0.95 | 0.95 | Yes | None |
| QA-005 | Bảo vệ quyền riêng tư | 1.00 | 0.79 | 1.00 | 0.93 | Yes | None |
| QA-013 | Feedback có cải thiện AI không? | 1.00 | 0.50 | 0.87 | 0.79 | Yes | None |
| QA-019 | Có thay đổi ngưỡng cảnh báo không? | 1.00 | 0.70 | 1.00 | 0.90 | Yes | None |
| QA-010 | HIGH/CRITICAL không phản hồi | 1.00 | 0.71 | 1.00 | 0.90 | Yes | None |
| QA-016 | Edge device dùng Firebase Token không? | 1.00 | 0.73 | 0.83 | 0.85 | Yes | None |
| QA-020 | Camera mất kết nối | 1.00 | 0.56 | 1.00 | 0.85 | Yes | None |

**Aggregate Report:**
- Overall pass rate: 100%
- Avg Faithfulness: 1.00
- Avg Relevance: 0.70
- Avg Completeness: 0.96
- Failure type distribution: {}

**3 câu hỏi scored thấp nhất:**
1. ID: QA-013 | Score: 0.79 | Failure type: None
2. ID: QA-006 | Score: 0.82 | Failure type: None
3. ID: QA-012 | Score: 0.83 | Failure type: None

**Benchmark note:** Kết quả này dùng retrieval-only baseline: agent demo chỉ trả lời bằng context trong golden dataset, không nhìn expected answer. Pass rate 100% nghĩa là tất cả câu vượt ngưỡng tối thiểu 0.5, nhưng relevance và completeness vẫn dao động nên không phải điểm tuyệt đối giả.

**Strict quality gate check (production-style):**
- Thresholds: Faithfulness >= 0.70, Relevance >= 0.70, Completeness >= 0.80
- Passed: 13/20
- Pass rate: 65%
- Weak cases cần review: QA-002, QA-006, QA-007, QA-012, QA-015, QA-013, QA-020

Ghi chú: Faithfulness đạt 1.00 vì baseline trả lời trực tiếp từ context, nên mọi token trong answer đều grounded. Điều này không đồng nghĩa agent đã đủ tốt cho production; relevance thấp hơn cho thấy cần semantic judge hoặc threshold nghiêm hơn.
---

### Exercise 3.3 — LLM-as-Judge Rubric Design

Theo bài giảng, rubric scoring 1–5 cần tiêu chí CỤ THỂ cho mỗi mức.

**Thiết kế rubric cho domain của bạn:**

| **Score** | **Tiêu chí (domain-specific)** | **Ví dụ response (Tình huống: Cảm biến báo té ngã tại vườn)** | **Ví dụ response (mẫu)** |
| --- | --- | --- | --- |
| **5** | **Hoàn hảo**: Xác định đúng sự cố, vị trí và thời gian. Đưa ra hành động khẩn cấp ngay lập tức theo giao thức. Giọng điệu bình tĩnh, rõ ràng. | Phát hiện té ngã rõ ràng, vị trí và thời gian chính xác; khởi động quy trình cấp cứu và liên hệ người thân ngay. |"Phát hiện té ngã tại vườn lúc 08:15. Tôi đã kích hoạt báo động tại chỗ và đang kết nối cuộc gọi với con gái bạn. Bạn có nghe thấy tôi không?" |
| **4** | **Tốt**: Thông tin chính xác và trung thực với dữ liệu cảm biến. Có hành động hỗ trợ nhưng thiếu một chi tiết nhỏ (ví dụ: không nhắc vị trí cụ thể). | Thông báo chính xác, có hành động hỗ trợ nhưng thiếu chi tiết phụ. | "Hệ thống ghi nhận bạn vừa bị ngã. Tôi đang gọi cho người giám hộ của bạn ngay bây giờ. Hãy giữ bình tĩnh." |
| **3** | **Tạm ổn**: Nhận diện được sự cố nhưng hành động thụ động, chưa quyết liệt hoặc thiếu thông tin quan trọng để người ứng cứu ra quyết định. | Nhận diện sự cố nhưng chỉ đề xuất hành động chung chung, không chủ động kích hoạt quy trình. | "Có vẻ đã có một sự cố té ngã xảy ra. Bạn có cần tôi giúp gọi cho ai đó không?"|
| **2** | **Kém**: Phân loại sai mức độ nghiêm trọng hoặc đưa thông tin gây nhầm lẫn (ví dụ: nhầm vị trí). Vi phạm quy định an toàn cơ bản. | Thông tin sai vị trí hoặc sai mức độ, có thể gây chậm trễ phản ứng. | "Phát hiện di chuyển bất thường tại phòng khách (thực tế là nhà tắm). Vui lòng kiểm tra lại thiết bị của bạn khi có thể." |
| **1** | **Nguy hiểm / Bịa đặt**: Hallucination hoặc khuyến nghị gây hại (ví dụ: khuyên không gọi cứu hộ khi cần). | Bịa đặt, làm giảm mức độ khẩn cấp hoặc đưa lời khuyên nguy hiểm. | "Bạn không sao đâu, cảm biến chỉ nhầm lẫn thôi. Hãy cứ nằm nghỉ, tôi sẽ tắt các thông báo cảnh báo cho bạn." |

**Criteria dimensions (chọn 3–5 từ list hoặc tự thêm):**
- [x] Correctness (đúng sự thật?)
- [x] Completeness (đủ chi tiết?)
- [x] Relevance (trả lời đúng câu hỏi?)
- [ ] Citation (trích nguồn?)
- [x] Tone (giọng phù hợp context?)
- [x] Actionability (có thể hành động theo?)
- [x] Safety (không có harmful content?)

**3 edge cases khó score:**

| **Edge Case** | **Tại sao khó score** | **Cách xử lý trong rubric** |
| --- | --- | --- |
| **Té ngã giả (False Positive)** | Vật rơi hoặc điện thoại gây báo nhầm; báo động quá mức gây alert fatigue, bỏ qua có thể vi phạm an toàn. | Nếu Agent có **bước xác nhận nhanh** (vòng lặp phản hồi) trước khi gọi cấp cứu, cho điểm 4–5; nếu không, giảm điểm theo mức độ gây phiền nhiễu/ rủi ro. |
| **Sự cố phức hợp (Multi-step)** | Người ngã kèm dấu hiệu bệnh lý (ví dụ đau tim); Agent có thể xử lý tốt té ngã nhưng bỏ sót dấu hiệu bệnh lý. | Yêu cầu chấm theo **Trajectory**: Agent phải duy trì hội thoại và cập nhật trạng thái; nếu có theo dõi liên tục và điều chỉnh hành động, cho điểm cao hơn. |
| **Yêu cầu ngoài phạm vi (Out-of-scope)** | Người già hỏi về đơn thuốc hoặc tư vấn y tế trong lúc khẩn cấp; khó phân định giữa trả lời hữu ích và vi phạm ranh giới y tế. | Thiết lập **Guardrail Breach = Score 1** nếu Agent trả lời vượt ranh giới y tế; nếu từ chối khéo và quay lại nhiệm vụ an toàn, cho điểm theo hành vi an toàn (>=3). |
---

### Exercise 3.4 — Framework Comparison (Bonus)

Nếu đã hoàn thành 3.1–3.3, chọn 2 trong 3 frameworks để so sánh:

| Aspect | **Framework 1: RAGAS** | **Framework 2: LLM-as-Judge (Multi-Judge/PoLL)** |
| --- | --- | --- |
| **Setup complexity** | **Trung bình**: Yêu cầu chuẩn bị dữ liệu theo schema cụ thể (SingleTurnSample gồm: user_input; response; context; reference). Cần pin đúng version **v0.4.3** để tránh vỡ code do API thay đổi. | **Cao**: Cần thiết kế rubric chi tiết, chọn tổ hợp các model "khác họ" (jury), thiết lập engine xử lý xung đột (Consensus Engine) và hiệu chuẩn (calibrate) với chuyên gia con người. |
| **Metrics available** | **Chuyên sâu RAG & Agent**: Tập trung vào 4 chỉ số cốt lõi (Faithfulness; Answer Relevancy; Context Precision/Recall) và các chỉ số Agent (ToolCallAccuracy; GoalAccuracy). | **Đa chiều & Linh hoạt**: Chấm điểm 1–5 hoặc so sánh cặp (Pairwise) dựa trên bất kỳ tiêu chí nào (Tone; Safety; Correctness) thông qua prompt tùy chỉnh. |
| **CI/CD integration** | **Rất tốt**: Thường dùng làm "Release Gate". Nếu điểm (ví dụ Faithfulness) < threshold (0.7) thì tự động chặn deploy. | **Tốt**: Tích hợp qua công cụ như Promptfoo hoặc Braintrust để thực hiện "shadow test" hoặc "canary rollout" và so sánh delta điểm với baseline. |
| **Score cho cùng dataset** | **Dựa trên Claim**: Điểm tính theo tỉ lệ các xác nhận (claims) được hỗ trợ bởi ngữ cảnh. | **Dựa trên Preference**: Điểm phản ánh sự đồng thuận với con người (ví dụ GPT‑4 đạt ~84% đồng thuận với chuyên gia). |
| Insight rút ra |RAGAS giúp xác định chính xác tầng bị hỏng (ví dụ Retrieval vs Generation) để fix đúng chỗ. |LLM-as-Judge giúp đánh giá các trục hành vi tinh tế (ví dụ tính nịnh bợ — sycophancy) mà các metric cứng không đo được. |

**Câu hỏi phân tích:**
- Scores có consistent giữa 2 frameworks không?
  > Không hoàn toàn. RAGAS-inspired heuristic nhất quán khi câu trả lời và context có overlap từ vựng rõ ràng, nhưng có thể đánh thấp câu trả lời đúng nếu dùng từ đồng nghĩa. LLM-as-Judge thường linh hoạt hơn với diễn đạt tự nhiên, nhưng dễ bị bias nếu rubric không chặt.
- Framework nào strict hơn? Tại sao?
  > RAGAS strict hơn ở tầng grounding/retrieval vì nó yêu cầu claim phải được hỗ trợ bởi context. LLM-as-Judge strict hay dễ phụ thuộc vào rubric, model judge và cách calibrate với human.
- Failure cases có giống nhau không?
  > Một phần giống nhau. Các lỗi hallucination hoặc thiếu evidence thường bị cả hai bắt. Tuy nhiên RAGAS dễ gắn lỗi vào retrieval/context, còn LLM-as-Judge dễ phát hiện lỗi tone, safety, actionability hoặc câu trả lời mơ hồ.

---

### Exercise 3.5 — Tăng Context Precision bằng Reranking (Nâng cao)

> **Bối cảnh:** Hai metrics retrieval — **Context Recall** và **Context Precision** —
> chấm điểm bước *get context* (retriever), chạy trên một **danh sách chunk**
> (`QAPair.retrieved_contexts`), không phải chuỗi context đơn.
>
> - **Context Recall** = `|expected ∩ (⋃ chunks)| / |expected|` — retriever có *lấy đủ* evidence không?
> - **Context Precision** = rank-aware Average Precision — chunk *relevant* có được *xếp lên đầu* không?
>
> Vì Precision tính theo thứ hạng (AP@K), **đổi thứ tự** chunk (đưa relevant lên trước)
> sẽ tăng điểm mà **không cần đổi tập chunk** → đó chính là việc của **reranking**.

#### Bước 1 — Dataset retrieval (đã cho sẵn để bạn chấm 2 metrics)

Mỗi dòng là 1 truy vấn với danh sách chunk retrieve được (cố tình để **noise lên trước**):

| ID | Question | Expected Answer | Retrieved chunks (theo thứ tự retriever trả về) |
|----|----------|-----------------|--------------------------------------------------|
| R01 | What is the capital of France? | Paris is the capital of France | `["Bananas are a tropical fruit.", "The Eiffel Tower is in Paris.", "Paris is the capital city of France."]` |
| R02 | What does RAG stand for? | RAG stands for Retrieval-Augmented Generation | `["LLMs can hallucinate facts.", "Retrieval-Augmented Generation (RAG) combines retrieval with generation.", "Vector databases store embeddings."]` |
| R03 | When was the Eiffel Tower built? | The Eiffel Tower was completed in 1889 | `["The tower is 330 metres tall.", "It is made of wrought iron.", "The Eiffel Tower was completed in 1889 for the World's Fair."]` |
| R04 | What is gradient descent? | Gradient descent minimizes a loss function by following the negative gradient | `["Neural networks have layers.", "Gradient descent updates weights along the negative gradient to minimize loss.", "Learning rate controls step size."]` |
| R05 | What is overfitting? | Overfitting is when a model memorizes training data and fails to generalize | `["Regularization adds a penalty term.", "Dropout randomly disables neurons.", "Overfitting means the model memorizes training data and generalizes poorly."]` |

> Bạn có thể tự thêm 3–5 dòng từ **domain của bạn** (Exercise 3.1) — nhớ để chunk relevant **không** ở vị trí đầu.

#### Bước 2 — Đo baseline (chưa rerank)

Với mỗi truy vấn, gọi:
```python
ev = RAGASEvaluator()
recall    = ev.evaluate_context_recall(chunks, expected)
precision = ev.evaluate_context_precision(chunks, expected)
```

| ID | Context Recall | Context Precision (before) |
|----|----------------|----------------------------|
| R01 | 1.00 | 0.58 |
| R02 | 0.80 | 0.50 |
| R03 | 1.00 | 0.83 |
| R04 | 0.57 | 0.50 |
| R05 | 0.62 | 0.33 |
| **Avg** | 0.80 | 0.55 |

#### Bước 3 — Rerank rồi đo lại

```python
reranked  = rerank_by_overlap(chunks, question)   # hoặc reranker bạn tự viết
precision = ev.evaluate_context_precision(reranked, expected)
```

| ID | Precision (before) | Precision (after rerank) | Δ |
|----|--------------------|--------------------------|---|
| R01 | 0.58 | 0.83 | +0.25 |
| R02 | 0.50 | 1.00 | +0.50 |
| R03 | 0.83 | 1.00 | +0.17 |
| R04 | 0.50 | 1.00 | +0.50 |
| R05 | 0.33 | 1.00 | +0.67 |
| **Avg** | 0.55 | 0.97 | +0.42 |

#### Bước 4 — Câu hỏi phân tích

1. **Recall có đổi sau khi rerank không? Tại sao?**
   > Không đổi. Rerank chỉ thay đổi thứ tự các chunk đã retrieve, không thêm hoặc bớt evidence. Vì Context Recall được tính trên union của toàn bộ chunks, tập token vẫn giống nhau nên recall giữ nguyên.

2. **Precision tăng bao nhiêu? Vì sao reranking lại tác động đúng vào precision chứ không phải recall?**
   > Precision trung bình tăng từ 0.55 lên 0.97, tức tăng +0.42. Context Precision dùng Average Precision theo thứ hạng, nên chunk relevant được đưa lên đầu sẽ làm Precision@K tăng. Recall không tăng vì reranking không lấy thêm evidence mới.

3. **Khi nào cần tăng Recall thay vì Precision?** (gợi ý: recall thấp = retriever bỏ sót evidence → rerank vô dụng, phải sửa retriever)
   > Cần tăng Recall khi evidence cần thiết hoàn toàn không nằm trong retrieved chunks. Khi đó rerank không giúp được vì không có chunk đúng để đưa lên đầu; cần sửa retriever bằng cách tăng top-k, hybrid search, query expansion, re-index dữ liệu hoặc điều chỉnh chunking.

#### Bước 5 — Kỹ thuật get-context để tăng điểm (chọn ≥ 3, mô tả tác động lên Recall vs Precision)

| Kỹ thuật | Tác động chính | Recall hay Precision? | Ghi chú triển khai |
|----------|----------------|-----------------------|--------------------|
| **Reranking** (cross-encoder, ví dụ `bge-reranker`, Cohere Rerank) | Xếp lại chunk theo độ liên quan | **Precision** ↑ | Retrieve dư (top-50) rồi rerank còn top-5 |
| **Tăng top-k khi retrieve** | Lấy nhiều chunk hơn | **Recall** ↑ (Precision có thể ↓) | Cân bằng với reranking |
| **Hybrid search** (BM25 + vector) | Bắt cả keyword lẫn semantic | Recall ↑ | Kết hợp lexical + dense |
| **Query rewriting / expansion** | Mở rộng truy vấn | Recall ↑ | HyDE, multi-query |
| **Chunk size / overlap tuning** | Giảm phân mảnh evidence | Recall + Precision | Chunk quá nhỏ → recall ↓ |
| **Metadata filtering** | Loại chunk sai domain/thời gian | Precision ↑ | Lọc trước khi rank |
| **MMR (Maximal Marginal Relevance)** | Giảm chunk trùng lặp | Precision ↑ | Đa dạng hoá kết quả |

**Pipeline khuyến nghị để tối ưu Precision (mô tả 1 đoạn):**
> Với SilentGuard AI, pipeline nên là: rewrite query để chuẩn hoá các thuật ngữ như "ngã/té ngã/cảnh báo", retrieve top-30 bằng hybrid search BM25 + vector, lọc metadata theo source_doc hoặc household_id nếu có, rerank bằng cross-encoder hoặc lexical overlap, giữ top-5 chunks liên quan nhất, sau đó dùng MMR để giảm trùng lặp trước khi đưa context vào generator. Cách này ưu tiên Precision nhưng vẫn giữ Recall đủ cao nhờ retrieve dư ở bước đầu.

#### (Tuỳ chọn) Bước 6 — Viết reranker của riêng bạn

Mặc định `rerank_by_overlap` chỉ dùng word-overlap. Hãy thử cải tiến (ví dụ: ưu tiên
chunk phủ nhiều token *expected* hơn, hoặc phạt chunk quá dài) và đo lại precision.

---

## Part 4 — Reflection (2:20–2:50)
See `reflection.md`

---

## Submission Checklist
- [x] All tests pass: `pytest tests/ -v`
- [x] `overall_score` implemented
- [x] `run_regression` implemented  
- [x] `generate_improvement_log` implemented
- [x] `evaluate_context_recall` + `evaluate_context_precision` implemented (Task 2b)
- [x] Exercise 3.5 completed: đo Context Recall/Precision + reranking before/after
- [x] `exercises.md` completed: golden dataset 20 QA (stratified) + benchmark results + rubric
- [ ] `reflection.md` written: 3 failures with 5 Whys + improvement log + CI/CD strategy
- [x] `solution/solution.py` copied
