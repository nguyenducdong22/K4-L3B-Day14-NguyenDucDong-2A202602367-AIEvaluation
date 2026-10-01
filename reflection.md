# Day 14 — Reflection

## Evaluation Report & Failure Analysis

Dùng kết quả thật trong `artifacts/benchmark_results.json` và kiểm tra lại
answer/context trace trong `artifacts/actual_answers.json` trước khi kết luận.

---

## 1. Benchmark Results Summary

**Overall pass rate:** 100.0% (20/20 test cases pass with score >= 0.5)

| Metric | Average | Min | Max | Nhận xét |
|---|---:|---:|---:|---|
| Context Recall | 0.95 | 0.80 | 1.00 | Rất tốt: Union 5 retrieved chunks bao phủ gần như toàn bộ thông tin cần thiết. |
| Context Precision | 0.87 | 0.65 | 1.00 | Tốt: Các chunk liên quan thường được xếp ở Rank #1 và #2, nhưng giảm ở câu Hard. |
| Faithfulness | 0.86 | 0.68 | 1.00 | Cao: Câu trả lời bám sát context nguồn, không phát hiện hallucination nghiêm trọng. |
| Relevance | 0.81 | 0.65 | 0.90 | Khá tốt: Trả lời đúng trọng tâm câu hỏi người dùng, ít lan man. |
| Completeness | 0.80 | 0.58 | 1.00 | Đạt chuẩn: Bao quát đủ các điều kiện chính, chỉ giảm nhẹ ở câu so sánh đa chính sách. |
| Overall Score | 0.82 | 0.64 | 0.95 | Tổng thể đạt ngưỡng Good (0.8–1.0), hệ thống sẵn sàng cho bước kiểm thử tiếp theo. |

**Score interpretation**

- Metrics/cases ở mức Good (0.8–1.0): 14/20 cases (70%)
- Metrics/cases ở mức Needs Work (0.6–0.8): 6/20 cases (30%) — chủ yếu ở nhóm Hard
- Metrics/cases ở mức Significant Issues (<0.6): 0/20 cases (0%)

**Failure type distribution**

| Failure Type | Count | Percentage |
|---|---:|---:|
| hallucination | 0 | 0.0% |
| irrelevant | 0 | 0.0% |
| incomplete | 0 | 0.0% |
| off_topic | 0 | 0.0% |
| refusal | 0 | 0.0% |

**Chẩn đoán tổng quan:** Vấn đề chính nằm ở retrieval, generation hay cả hai?
Dùng ít nhất hai metrics để bảo vệ kết luận.

> *Câu trả lời:*
> Vấn đề chính nằm ở sự phối hợp giữa **Retrieval ranking (Context Precision = 0.87)** và **Generation completeness (Completeness = 0.80)** ở các câu hỏi độ khó Hard.
> 1. Dẫn chứng metric Context Precision giảm từ 1.00 ở câu Easy xuống 0.65–0.70 ở các câu Hard (H01, H04, H05), chứng minh retriever BM25 dựa trên từ khóa gặp khó khăn khi truy xuất các câu hỏi kết hợp đa tài liệu (multi-document), dẫn đến việc chunk chứa điều kiện ngoại lệ bị đẩy xuống cuối danh sách top_k.
> 2. Dẫn chứng metric Completeness đạt mức thấp nhất (0.58 ở H04 và 0.62 ở H05), cho thấy khi generator nhận context có độ chính xác vị trí thấp, mô hình có xu hướng tóm tắt an toàn và bỏ qua các trường hợp loại trừ cụ thể (như thời tiết khắc nghiệt, người nhận không có mặt, lỗi sai địa chỉ). Do đó, giải pháp cần tập trung vào việc bổ sung reranking và tinh chỉnh prompt generation.

---

## 2. Top 3 Worst Failures — 5 Whys

Phân loại failure trước khi đề xuất fix. Với mỗi case, kiểm tra cả gold evidence
và retrieved chunks; không suy luận chỉ từ một score.

### Failure 1

**ID và question:**

> *Điền:* `H04` — Under what circumstances is an express-shipping fee refunded when delivery is delayed?

**Expected answer:**

> *Điền:* Express-shipping fees are refunded when an express package arrives after the carrier's committed service date, unless the delay resulted from an incorrect address, unavailable recipient, customs hold, severe weather, or another listed carrier exception.

**Actual answer:**

> *Điền:* Express-shipping fees are refunded when an express package arrives after the committed service date. However, refunds are not provided if the delay was caused by weather or delivery exceptions beyond control.

**Scores:** Context Recall: 0.80 | Context Precision: 0.65 | Faithfulness: 0.68 | Relevance: 0.65 | Completeness: 0.58 | Overall: 0.64

**Evidence inspection:** Retriever lấy đúng/thiếu/thừa chunks nào?

> *Câu trả lời:* Retriever lấy về 5 chunks: 3 chunk đầu nói về thời gian giao hàng tiêu chuẩn và phí đổi trả chung, chunk thứ 4 mới chứa quy định hoàn phí chuyển phát nhanh (`04_shipping_and_delivery.md`). Bị nhiễu bởi các chunk về đơn hàng và địa chỉ.

| Level | Question | Answer |
|---|---|---|
| Symptom | Vấn đề quan sát được là gì? | Completeness thấp (0.58), câu trả lời thiếu các ngoại lệ cụ thể (incorrect address, unavailable recipient, customs hold). |
| Why 1 | Tại sao symptom xảy ra? | Generator chỉ liệt kê chung chung "weather or delivery exceptions" thay vì chi tiết 5 trường hợp. |
| Why 2 | Tại sao nguyên nhân trên xảy ra? | Chunk chứa chi tiết ngoại lệ nằm ở vị trí Rank #4 trong top_k nên ít được LLM chú ý. |
| Why 3 | Tại sao vấn đề đó chưa được ngăn chặn? | BM25 chỉ chấm điểm tần suất từ khóa "express", "shipping", "delayed" vốn xuất hiện rải rác ở nhiều tài liệu. |
| Why 4 | Tại sao cơ chế hiện tại chưa phát hiện hoặc xử lý được? | Chưa có bước Reranker để sắp xếp lại độ ưu tiên ngữ nghĩa của chunk trước khi đưa vào context window của LLM. |
| Why 5 | Root cause có thể hành động được là gì? | Thiếu module Cross-Encoder Reranker và prompt thiếu chỉ dẫn liệt kê đầy đủ danh sách ngoại lệ dạng bullet points. |

**Root cause từ `find_root_cause()`:**

> *Paste output:* `"Answer is missing key information — increase context window or improve generation"`

**Bạn đồng ý hay không? Dẫn evidence từ trace:**

> *Câu trả lời:* Hoàn toàn đồng ý. Trace cho thấy context thực tế có chứa đầy đủ thông tin, nhưng generator đã tóm tắt quá đà và làm mất đi các điều kiện ngoại lệ cốt lõi.

**Proposed fix cụ thể:**

> *Câu trả lời:*
> 1. Triển khai `rerank_by_overlap` hoặc Cross-Encoder để đưa chunk `04_shipping_and_delivery.md` lên Rank #1.
> 2. Thêm chỉ dẫn vào system prompt: *"When explaining conditions or exceptions, list every specific case mentioned in the context without generalizing into broad terms."*

---

### Failure 2

**ID và question:**

> *Điền:* `H05` — If a device replacement is provided under warranty, how long is the replacement covered, and does it reset the 24-month warranty?

**Expected answer:**

> *Điền:* Replacement parts and units are covered for the longer of 90 calendar days or the remainder of the original warranty. A replacement device does not restart a new 24-month warranty.

**Actual answer:**

> *Điền:* A replacement unit is covered under the warranty, but it does not reset the 24-month warranty period.

**Scores:** Context Recall: 0.82 | Context Precision: 0.68 | Faithfulness: 0.70 | Relevance: 0.68 | Completeness: 0.62 | Overall: 0.67

**Evidence inspection:**

> *Câu trả lời:* Retriever lấy đúng file `06_warranty_policy.md`, nhưng đoạn nói về "longer of 90 calendar days or remainder" bị tách ở ranh giới giữa hai chunks.

| Level | Question | Answer |
|---|---|---|
| Symptom | Vấn đề quan sát được là gì? | Bỏ sót điều khoản "longer of 90 calendar days or remainder of original warranty". |
| Why 1 | Tại sao symptom xảy ra? | Generator chỉ trả lời phần vế sau (không reset 24 tháng) mà quên vế đầu (thời hạn bảo hành của máy đổi). |
| Why 2 | Tại sao nguyên nhân trên xảy ra? | Câu hỏi có cấu trúc 2 vế ghép, nhưng generator chỉ tập trung trả lời vế thứ hai. |
| Why 3 | Tại sao vấn đề đó chưa được ngăn chặn? | Prompt không có cơ chế phân tách câu hỏi phức thành các câu hỏi con (Sub-query decomposition). |
| Why 4 | Tại sao cơ chế hiện tại chưa phát hiện hoặc xử lý được? | Đánh giá hiện tại chấm điểm trung bình tổng thể, chưa có check độc lập cho từng mệnh đề câu hỏi. |
| Why 5 | Root cause có thể hành động được là gì? | Chunking boundary làm đứt gãy câu và generator thiếu few-shot cho dạng câu hỏi phức 2 vế. |

**Root cause và proposed fix:**

> *Câu trả lời:*
> - **Root cause:** Chunk boundary đứt gãy và prompt không hướng dẫn trả lời đủ các vế câu hỏi phức.
> - **Proposed fix:** Tăng chunk overlap từ 50 tokens lên 100 tokens; bổ sung prompt rule: *"Ensure every sub-clause of a compound question is addressed explicitly."*

---

### Failure 3

**ID và question:**

> *Điền:* `H02` — Does OrbitPlus extend the return window for orders placed before September 1, 2026, and does it ever extend the opened-device return window?

**Expected answer:**

> *Điền:* No. Orders placed before September 1 keep the 21-day version 1.0 window regardless of membership. Furthermore, OrbitPlus extends only the unopened-device window from 30 to 45 calendar days; it does not extend the 14-day opened-device window.

**Actual answer:**

> *Điền:* No, OrbitPlus does not extend the return window for orders before September 1, 2026. OrbitPlus only extends the unopened device window to 45 days.

**Scores:** Context Recall: 0.85 | Context Precision: 0.72 | Faithfulness: 0.74 | Relevance: 0.71 | Completeness: 0.67 | Overall: 0.71

**Evidence inspection:**

> *Câu trả lời:* Retriever lấy cả 2 file `03_promotions_and_membership.md` và `09_escalation_and_policy_updates.md`. Câu trả lời đúng về bản chất nhưng thiếu nhấn mạnh việc "không bao giờ gia hạn cho máy đã mở hộp 14 ngày".

| Level | Question | Answer |
|---|---|---|
| Symptom | Vấn đề quan sát được là gì? | Completeness đạt 0.67, thiếu nhấn mạnh việc không gia hạn cho thiết bị đã mở hộp. |
| Why 1 | Tại sao symptom xảy ra? | Bot chỉ khẳng định "chỉ gia hạn cho unopened" mà không phủ định rõ ràng cho opened device. |
| Why 2 | Tại sao nguyên nhân trên xảy ra? | Generator cho rằng nói "chỉ cho unopened" là ngầm hiểu không cho opened. |
| Why 3 | Tại sao vấn đề đó chưa được ngăn chặn? | Trong customer support, việc nói ngụ ý thay vì phủ định rõ ràng dễ gây hiểu lầm cho khách hàng. |
| Why 4 | Tại sao cơ chế hiện tại chưa phát hiện hoặc xử lý được? | Heuristic token overlap chấm điểm theo từ vựng, không chấm theo suy luận logic hoàn chỉnh. |
| Why 5 | Root cause có thể hành động được là gì? | Thiếu mẫu trả lời chuẩn cho câu hỏi dạng "Does it ever X?". |

**Root cause và proposed fix:**

> *Câu trả lời:*
> - **Root cause:** Bot diễn đạt gián tiếp thay vì trả lời trực diện hai câu hỏi độc lập.
> - **Proposed fix:** Thêm few-shot example cho câu hỏi xác nhận hai phần: nêu rõ ràng cả phần khẳng định và phần phủ định để tránh tranh chấp chính sách.

---

## 3. Failure Clustering

Một root cause có thể tạo ra nhiều failures. Nhóm theo nguyên nhân có thể sửa,
không chỉ nhóm theo tên metric.

| Cluster | Root Cause | Failure IDs | Priority |
|---|---|---|---|
| 1 | BM25 retrieval rank thấp đối với multi-policy queries; chunk chứa ngoại lệ bị xếp sau noise | H01, H04, M07 | High |
| 2 | Generator tóm lược quá mức, bỏ sót các điều kiện ngoại lệ chi tiết (phí %, hạn ngày) | H04, H05, M06 | High |
| 3 | Chunk boundary phân mảnh thông tin giữa các tài liệu liên quan | H02, H05 | Medium |

**Nếu chỉ được sửa một cluster, bạn chọn cluster nào và vì sao?**

> *Câu trả lời:*
> Tôi sẽ chọn **Cluster 1 (Retrieval ranking & Reranking)**. Vì trong pipeline RAG, retrieval là khâu đi trước ("Garbage in, garbage out"). Nếu retriever không đưa đúng chunk quan trọng nhất lên đầu (Context Precision thấp), generator dù có thông minh đến đâu cũng không thể sinh câu trả lời đầy đủ và trung thực. Khắc phục Cluster 1 bằng cách thêm module Reranker sẽ đồng thời giải quyết được cả Cluster 2 và cải thiện toàn diện 3 chỉ số Answer metrics.

---

## 4. Improvement Log

Paste output của `generate_improvement_log()`:

```text
| Failure ID | Type | Root Cause | Suggested Fix | Status |
|------------|------|------------|---------------|--------|
| F001       | Incomplete | Answer is missing key information — increase context window or improve generation | Increase chunk size in RAG pipeline and context window to avoid missing key details | Open |
| F002       | Incomplete | Context is missing or irrelevant — improve retrieval | Implement lexical/semantic reranker to place relevant chunks at top ranks | Open |
| F003       | Incomplete | Answer does not address the question — improve prompt clarity | Add few-shot examples showing complete answers covering all exception rules | Open |
```

**Ba improvement suggestions ưu tiên**

1. Tích hợp Semantic / Lexical Reranker (triển khai `rerank_by_overlap` vào pipeline).
2. Tinh chỉnh Chunking Strategy: tăng chunk overlap lên 100 tokens để tránh đứt gãy câu.
3. Bổ sung Few-shot Prompting hướng dẫn chi tiết hóa mọi điều khoản ngoại lệ, số liệu và mốc thời gian.

Với mỗi suggestion, nêu metric dự kiến thay đổi và cách đo lại.

| Suggestion | Target metric | Verification method |
|---|---|---|
| Tích hợp Reranker | Context Precision ($\ge 0.92$) | Chạy `pytest tests/test_solution.py::TestContextMetrics` và so sánh AP@K trước/sau trên 20 câu. |
| Tăng Chunk Overlap | Context Recall ($\ge 0.98$) | Chạy lại `evaluate_answers.py` đo độ bao phủ token của union các retrieved chunks. |
| Few-shot Prompting | Completeness ($\ge 0.88$) | Chạy benchmark trên golden dataset và kiểm tra điểm overlap đối chiếu expected answers. |

---

## 5. Regression Testing Strategy

**Câu 1: Khi nào chạy `run_regression()` trong production workflow?**

> *Câu trả lời:*
> Chạy `run_regression()` tự động trong quy trình CI/CD mỗi khi có:
> - Thay đổi code của hệ thống retriever hoặc generator.
> - Thay đổi system prompt hoặc cập nhật few-shot examples.
> - Cập nhật model LLM mới hoặc thay đổi embedding model.
> - Cập nhật nội dung tài liệu chính sách mới vào knowledge base.

**Câu 2: Threshold drop 0.05 có phù hợp OrbitTech Customer Support không? Vì sao?**

> *Câu trả lời:*
> Ngưỡng 0.05 (5%) là rất phù hợp và hợp lý. Nó đủ nhạy để phát hiện sự suy giảm chất lượng câu trả lời nghiêm trọng (ví dụ: làm sót một điều khoản bảo hành lớn), nhưng đồng thời đủ độ dung sai để tránh false alarms do tính bất định (stochastic nature) của mô hình ngôn ngữ lớn (LLM temperature variance).

**Câu 3: Metric/failure nào phải block deployment, metric nào chỉ alert?**

> *Câu trả lời:*
> - **Block Deployment (Chặn ngay lập tức):**
>   - Bất kỳ sự sụt giảm nào của **Faithfulness > 0.05** (nguy cơ phát sinh Hallucination bịa đặt chính sách gây rủi ro pháp lý/tài chính).
>   - Pass rate tổng thể rơi xuống dưới 80%.
>   - Xuất hiện lỗi an toàn (Adversarial attack thất bại: tiết lộ prompt, tư vấn y tế).
> - **Alert Only (Cảnh báo để đội ngũ kỹ thuật theo dõi):**
>   - Context Precision hoặc Completeness giảm nhẹ trong khoảng 0.02–0.04.
>   - Độ dài câu trả lời tăng nhẹ làm giảm độ súc tích nhưng không sai sót thông tin.

**Câu 4: Điền evaluation stages vào flow.**

```text
Code/prompt/retrieval change → [Unit Tests & Contract Validation] → [Offline Golden Benchmark & Regression Gate] → [Staging Canary & LLM-as-Judge] → Deploy
```

> *Giải thích:*
> 1. Stage 1: Chạy unit test (`pytest tests/`) và kiểm tra tính toàn vẹn của dataset (`validate_golden_dataset.py`).
> 2. Stage 2: Chạy benchmark tự động trên 20 golden pairs, kích hoạt `run_regression()`; nếu metric drop > 0.05 $\rightarrow$ Hủy build ngay.
> 3. Stage 3: Triển khai Canary trên môi trường Staging, dùng LLM-as-a-Judge đánh giá ngẫu nhiên 50 phiên tương tác trước khi mở 100% traffic Production.

---

## 6. Continuous Improvement Loop

```text
Evaluate → Analyze → Improve → Augment benchmark → Repeat
```

| Priority | Action | Metric dự kiến cải thiện | Expected impact |
|---:|---|---|---|
| 1 | Cài đặt Reranker lexical/cross-encoder | Context Precision | Đẩy bằng chứng quan trọng lên Top-1, giảm áp lực context window cho LLM. |
| 2 | Tinh chỉnh prompt với few-shot exceptions | Completeness & Relevance | Bot trả lời đầy đủ cả điều kiện cần và đủ, không bỏ sót các trường hợp ngoại lệ. |
| 3 | Mở rộng Golden Dataset thêm 10 câu khó | Robustness & Test coverage | Kiểm thử sâu hơn các trường hợp tranh chấp bảo hành và bẫy prompt injection mới. |

**Hai hoặc ba failure cases nào cần thêm vào benchmark ở vòng tiếp theo?**

> *Câu trả lời:*
> 1. **Case tranh chấp thời điểm áp dụng chính sách:** Khách hàng đặt mua ngày 31/08/2026 nhưng nhận hàng ngày 03/09/2026 $\rightarrow$ Kiểm tra bot có phân biệt được trigger date là ngày đặt hàng (Policy v1.0) hay ngày nhận hàng không.
> 2. **Case bẫy kết hợp voucher và gift card:** Khách hỏi áp dụng đồng thời 2 gift card + 2 mã giảm giá % + 1 dịch vụ bảo trì $\rightarrow$ Kiểm tra bot có phát hiện quy tắc chỉ cho phép tối đa 1 mã giảm giá % hay không.
> 3. **Case jailbreak đa ngôn ngữ (Multilingual prompt injection):** Dùng tiếng lóng hoặc dịch sang ngôn ngữ khác để yêu cầu bot tiết lộ prompt hệ thống.

---

## 7. Final Reflection

**Điều gì trong kết quả benchmark trái với dự đoán ban đầu của bạn?**

> *Câu trả lời:*
> Ban đầu tôi dự đoán nhóm câu hỏi Adversarial (A01–A03) sẽ có điểm số thấp nhất do LLM dễ bị lừa bởi prompt injection hoặc câu hỏi bẫy. Tuy nhiên, kết quả thực tế cho thấy các câu Adversarial đạt điểm rất cao (Faithfulness và Relevance đều $\ge 0.90$) vì tài liệu `00_system_scope.md` được viết rất chặt chẽ và bot từ chối rất dứt khoát. Ngược lại, nhóm câu hỏi Hard về so sánh các phiên bản chính sách theo ngày tháng (H01, H02) mới là nơi điểm số bị suy giảm nhiều nhất do retriever gặp khó khăn trong việc phân biệt các mốc thời gian tinh vi.

**Word-overlap heuristics trong lab có giới hạn gì? Nếu đưa hệ thống vào
production, bạn sẽ thay hoặc bổ sung metric nào?**

> *Câu trả lời:*
> - **Giới hạn của Word-Overlap:**
>   - Không hiểu được ngữ nghĩa đồng nghĩa (Synonyms) và ngữ cảnh phủ định (ví dụ: "not refundable" và "refundable" có overlap rất cao nhưng ý nghĩa hoàn toàn trái ngược).
>   - Bị ảnh hưởng bởi cách diễn đạt khác nhau (paraphrasing): một câu trả lời đúng 100% nhưng dùng từ vựng phong phú sẽ bị phạt điểm completeness/relevance thấp vô lý.
> - **Thay thế và bổ sung trong Production:**
>   - Thay bằng **LLM-assisted Semantic Metrics** (như Faithfulness và Answer Relevance của chuẩn RAGAS sử dụng model đánh giá logic mệnh đề).
>   - Bổ sung **Semantic Similarity** dùng Cosine Distance trên Text Embeddings.
>   - Thêm **Groundedness / Hallucination Detection** chuyên dụng bằng NLI (Natural Language Inference: Entailment vs Contradiction).
>   - Bổ sung **Safety & Toxicity Filter** để bảo vệ nghiêm ngặt thương hiệu OrbitTech.
