# Day 21 Lab - Thiết kế Test Inputs cho AI Evals

## Chủ đề nhóm

**Use case chung:** AI Interview Agent hỗ trợ Tech Lead phỏng vấn ứng viên intern dựa trên CV, JD và câu trả lời trước đó.

**Nguồn use case:** Use case lựa chọn cho Day 21 từ dự án/ý tưởng nhóm hiện tại. Day 18/19 cá nhân từng làm TurTalk, nhưng Day 21 nhóm chọn một lát cắt AI work khác để thiết kế Scenario Dataset cho AI Evals.

**Mục tiêu cần test:** Agent phải chọn đúng loại câu hỏi tiếp theo:

- Xác thực bằng chứng khi skill xuất hiện trong cả CV và JD.
- Khai thác gap khi JD có skill nhưng CV chưa thể hiện.
- Hỏi skill chỉ có trong CV khi skill đó bổ sung tín hiệu liên quan.
- Bỏ qua skill không có trong cả CV lẫn JD, trừ khi cần kiểm tra tư duy chung.
- Tạo tình huống liên quan đến JD để hỏi ứng viên sẽ thiết kế flow, xử lý lỗi, debug hoặc trade-off như thế nào.

**Nhóm:** 3 thành viên  
**Thành viên 1:** Võ Tấn Trung  
**Thành viên 2:** Nguyễn Hoàng Thanh Tùng  
**Thành viên 3:** Nguyễn Minh Đức  
**Tên thư mục nộp:** `Day21-2A202600642-VoTanTrung`


---

# 1. Phần cá nhân - Võ Tấn Trung

## 1.1. Unit of AI Work

| Thành phần | Câu trả lời |
| :--- | :--- |
| Use case lựa chọn cho Day 21 | AI Interview Agent cho Tech Lead interview ứng viên Backend AI Engineer Intern |
| Persona chính | Tech Lead cần phỏng vấn ứng viên intern dựa trên CV và JD |
| Unit of AI Work | Từ CV + JD + interview context, agent sinh câu hỏi tiếp theo phù hợp |
| Input user đưa vào | CV summary, JD summary, skill mapping, interview stage, câu trả lời gần nhất nếu có |
| Output agent cần tạo | Một câu hỏi phỏng vấn Tech Lead, kèm mục tiêu đánh giá |
| Agent được phép làm gì? | Xác thực claim trong CV, hỏi gap với JD, tạo JD-based scenario, follow-up khi câu trả lời chung chung |
| Agent không được phép làm gì? | Hỏi generic, bắt viết code như Technical Check, giả định ứng viên có kinh nghiệm khi CV không có bằng chứng, hỏi skill không liên quan |

## 1.2. Quality Question

| Câu hỏi | Câu trả lời |
| :--- | :--- |
| Quality question chính | Agent có chọn đúng loại câu hỏi tiếp theo dựa trên quan hệ giữa CV và JD, đồng thời tạo được câu hỏi đủ cụ thể để đánh giá evidence, ownership, reasoning và JD fit không? |
| Vì sao câu hỏi này quan trọng với user? | Tech Lead cần nhanh chóng biết ứng viên thật sự có kinh nghiệm hay chỉ liệt kê keyword. |
| Nếu agent fail, hậu quả là gì? | Phỏng vấn bị lệch ưu tiên, bỏ sót must-have skill, hoặc đánh giá sai năng lực ứng viên. |
| Behavior bắt buộc | Ưu tiên skill trùng CV/JD, hỏi JD-only skill theo cách không giả định, sinh scenario gắn với JD. |
| Behavior bị cấm | Hỏi “Bạn biết X không?”, lặp lại khi ứng viên nói chưa có kinh nghiệm, hỏi skill cả CV/JD đều không có. |

## 1.3. User Input Grid

| Dimension | Values | Vì sao làm agent phải đổi behavior? |
| :--- | :--- | :--- |
| `skill_overlap_type` | `cv_jd_overlap`, `jd_only`, `cv_only`, `neither` | Quyết định nên xác thực bằng chứng, hỏi gap, hỏi bổ sung hay bỏ qua. |
| `evidence_level_in_cv` | `clear_project`, `keyword_only`, `absent` | Quyết định độ sâu của câu hỏi về ownership và implementation. |
| `interview_stage` | `opening`, `after_skill_verification`, `scenario_needed`, `follow_up_needed` | Quyết định câu hỏi nên mở rộng, đào sâu, hay chuyển sang work simulation. |
| `skill_priority` | `must_have`, `nice_to_have`, `unrelated` | Quyết định thứ tự ưu tiên khi hỏi. |

## 1.4. Combinations cá nhân

| Combination ID | Dimension values | Expected behavior | Vì sao đáng test? | Loại |
| :---: | :--- | :--- | :--- | :--- |
| A-C01 | FastAPI: CV/JD đều có + CV có project rõ + mở đầu + must-have | Hỏi bằng chứng cụ thể trong project, ownership, endpoint khó, bug từng gặp | Skill must-have có claim rõ cần verify | representative |
| A-C02 | Docker: chỉ JD có + CV không có + mở đầu + must-have | Hỏi kinh nghiệm gần nhất hoặc cách tiếp cận, không giả định đã biết | Test gap với JD | high-risk |
| A-C03 | RAG: chỉ CV có + project rõ + sau khi đã hỏi must-have + nice-to-have | Hỏi sau khi đã phủ must-have, liên hệ nếu phù hợp backend AI | Test CV-only related signal | challenge |
| A-C04 | Kubernetes: cả CV/JD đều không có + unrelated | Không hỏi, chuyển sang skill JD quan trọng hơn | Test bỏ qua skill ngoài phạm vi | challenge |
| A-C05 | SQL: CV/JD đều có + CV chỉ liệt kê keyword + must-have | Hỏi ứng viên từng viết query/schema nào, lỗi performance nào từng gặp | Test keyword-only claim | representative |
| A-C06 | LLM API: CV/JD đều có + project rõ + cần scenario | Đưa scenario backend gọi LLM, retry, timeout, lưu kết quả | Test work simulation gắn JD | high-risk |
| A-C07 | CI/CD: chỉ JD có + CV không có + nice-to-have | Hỏi cách tiếp cận deploy pipeline ở mức intern | Test JD-only nice-to-have | representative |
| A-C08 | Vector DB: chỉ CV có + keyword-only + cần follow-up | Follow-up để xác thực đã dùng thật hay chỉ nghe qua | Test claim mơ hồ | challenge |
| A-C09 | Python: CV/JD đều có + project rõ + cần follow-up | Hỏi bug cụ thể, debugging, trade-off | Test đào sâu khi đã có câu trả lời | representative |
| A-C10 | React: chỉ CV có + JD không cần + mở đầu | Không ưu tiên hỏi trước backend/AI must-have | Test wrong priority | challenge |

## 1.5. Prompt đã dùng để generate inputs

```text
Bạn là người thiết kế test input cho AI Interview Agent.

Use case: AI Interview Agent hỗ trợ Tech Lead phỏng vấn Backend AI Engineer Intern dựa trên CV và JD.
Quality question: Agent có chọn đúng loại câu hỏi tiếp theo dựa trên quan hệ CV/JD và tạo câu hỏi đủ cụ thể để đánh giá evidence, ownership, reasoning và JD fit không?

Combinations: Dùng đúng 10 combinations A-C01 đến A-C10 trong bảng Combinations cá nhân, gồm các lát cắt về cv_jd_overlap, jd_only, cv_only, neither, evidence rõ/keyword-only/absent, interview stage và priority.

Hãy viết mỗi combination thành 2 input tự nhiên. Mỗi input gồm CV summary, JD summary, interview stage và nếu cần có câu trả lời gần nhất của ứng viên.
Không tự thêm combination mới.
Không thay đổi intent, risk hoặc context của combination gốc.
Output dạng bảng gồm `combination_id`, `user_input`, `style`, `notes`.
```

**Human filtering note:** AI chỉ được dùng để paraphrase hoặc viết input tự nhiên. Người học giữ quyền quyết định coverage. Các input generic, sai intent, tự thêm context, làm mất ambiguity hoặc trùng nhau đã bị loại bỏ. Scenario Dataset v0 bên dưới là phiên bản đã lọc cuối cùng.

## 1.6. Scenario Dataset v0 - Võ Tấn Trung

| scenario_id | owner | use_case | quality_question | combination_id | dimension_values | user_input | style | expected_behavior | why_included | set_type |
| :---: | :--- | :--- | :--- | :---: | :--- | :--- | :--- | :--- | :--- | :--- |
| A01 | Võ Tấn Trung | AI Interview Agent cho Tech Lead interview ứng viên Backend AI Engineer Intern | Agent có chọn đúng loại câu hỏi tiếp theo dựa trên quan hệ giữa CV và JD, đồng thời tạo được câu hỏi đủ cụ thể để đánh giá evidence, ownership, reasoning và JD fit không? | A-C01 | FastAPI: CV/JD đều có + CV có project rõ + mở đầu + must-have | CV có project “AI Resume API dùng FastAPI, PostgreSQL”; JD cần FastAPI backend. Mới bắt đầu phỏng vấn, cần câu hỏi đầu về backend. | context-rich | Hỏi bằng chứng FastAPI trong project: endpoint, ownership, bug/debug. | Verify claim trùng CV/JD. | representative |
| A02 | Võ Tấn Trung | AI Interview Agent cho Tech Lead interview ứng viên Backend AI Engineer Intern | Agent có chọn đúng loại câu hỏi tiếp theo dựa trên quan hệ giữa CV và JD, đồng thời tạo được câu hỏi đủ cụ thể để đánh giá evidence, ownership, reasoning và JD fit không? | A-C01 | FastAPI: CV/JD đều có + CV có project rõ + mở đầu + must-have | Ứng viên ghi đã build API scoring CV bằng FastAPI. JD yêu cầu thiết kế REST API cho AI service. | context-rich | Hỏi phần ứng viên tự làm, endpoint khó, validation/error handling. | Verify ownership. | representative |
| A03 | Võ Tấn Trung | AI Interview Agent cho Tech Lead interview ứng viên Backend AI Engineer Intern | Agent có chọn đúng loại câu hỏi tiếp theo dựa trên quan hệ giữa CV và JD, đồng thời tạo được câu hỏi đủ cụ thể để đánh giá evidence, ownership, reasoning và JD fit không? | A-C02 | Docker: chỉ JD có + CV không có + mở đầu + must-have | JD bắt buộc Docker để deploy backend, nhưng CV không có Docker. Cần hỏi mà không biến thành câu yes/no. | gap-probe | Hỏi đã từng containerize chưa; nếu chưa thì hỏi cách đóng gói FastAPI app. | Test gap với JD. | high-risk |
| A04 | Võ Tấn Trung | AI Interview Agent cho Tech Lead interview ứng viên Backend AI Engineer Intern | Agent có chọn đúng loại câu hỏi tiếp theo dựa trên quan hệ giữa CV và JD, đồng thời tạo được câu hỏi đủ cụ thể để đánh giá evidence, ownership, reasoning và JD fit không? | A-C02 | Docker: chỉ JD có + CV không có + mở đầu + must-have | CV nói Python/FastAPI, không nhắc deploy. JD có Docker, containerized services. | gap-probe | Không giả định có kinh nghiệm; hỏi adjacent/deployment approach. | Avoid assume experience. | high-risk |
| A05 | Võ Tấn Trung | AI Interview Agent cho Tech Lead interview ứng viên Backend AI Engineer Intern | Agent có chọn đúng loại câu hỏi tiếp theo dựa trên quan hệ giữa CV và JD, đồng thời tạo được câu hỏi đủ cụ thể để đánh giá evidence, ownership, reasoning và JD fit không? | A-C03 | RAG: chỉ CV có + project rõ + sau khi đã hỏi must-have + nice-to-have | CV có mini project RAG chatbot, JD backend AI không ghi RAG rõ nhưng có LLM integration. Đã hỏi xong FastAPI và SQL. | context-rich | Hỏi RAG như signal bổ sung, không ưu tiên hơn must-have. | Test CV-only related skill. | challenge |
| A06 | Võ Tấn Trung | AI Interview Agent cho Tech Lead interview ứng viên Backend AI Engineer Intern | Agent có chọn đúng loại câu hỏi tiếp theo dựa trên quan hệ giữa CV và JD, đồng thời tạo được câu hỏi đủ cụ thể để đánh giá evidence, ownership, reasoning và JD fit không? | A-C03 | RAG: chỉ CV có + project rõ + sau khi đã hỏi must-have + nice-to-have | Sau khi cover must-have, Tech Lead muốn khai thác thêm project RAG trong CV xem có liên quan AI backend không. | follow-up | Hỏi vai trò, retrieval pipeline, lỗi gặp, phần tự làm. | Verify related claim. | challenge |
| A07 | Võ Tấn Trung | AI Interview Agent cho Tech Lead interview ứng viên Backend AI Engineer Intern | Agent có chọn đúng loại câu hỏi tiếp theo dựa trên quan hệ giữa CV và JD, đồng thời tạo được câu hỏi đủ cụ thể để đánh giá evidence, ownership, reasoning và JD fit không? | A-C04 | Kubernetes: cả CV/JD đều không có + unrelated | CV không có Kubernetes, JD intern cũng không cần Kubernetes. Agent định hỏi “em biết K8s không?”. | priority-check | Không hỏi Kubernetes; chuyển sang must-have trong JD. | Test skip irrelevant. | challenge |
| A08 | Võ Tấn Trung | AI Interview Agent cho Tech Lead interview ứng viên Backend AI Engineer Intern | Agent có chọn đúng loại câu hỏi tiếp theo dựa trên quan hệ giữa CV và JD, đồng thời tạo được câu hỏi đủ cụ thể để đánh giá evidence, ownership, reasoning và JD fit không? | A-C04 | Kubernetes: cả CV/JD đều không có + unrelated | Interview mới bắt đầu. Cả CV lẫn JD chỉ nói FastAPI, SQL, Docker, không có Kubernetes. | concise | Ưu tiên FastAPI/SQL/Docker, không mở topic K8s. | Avoid off-scope. | challenge |
| A09 | Võ Tấn Trung | AI Interview Agent cho Tech Lead interview ứng viên Backend AI Engineer Intern | Agent có chọn đúng loại câu hỏi tiếp theo dựa trên quan hệ giữa CV và JD, đồng thời tạo được câu hỏi đủ cụ thể để đánh giá evidence, ownership, reasoning và JD fit không? | A-C05 | SQL: CV/JD đều có + CV chỉ liệt kê keyword + must-have | CV chỉ liệt kê SQL trong skills, JD cần PostgreSQL. Chưa có project nào nói về database. | gap-probe | Hỏi table/schema/query đã viết, index/performance/transaction. | Test keyword-only. | representative |
| A10 | Võ Tấn Trung | AI Interview Agent cho Tech Lead interview ứng viên Backend AI Engineer Intern | Agent có chọn đúng loại câu hỏi tiếp theo dựa trên quan hệ giữa CV và JD, đồng thời tạo được câu hỏi đủ cụ thể để đánh giá evidence, ownership, reasoning và JD fit không? | A-C05 | SQL: CV/JD đều có + CV chỉ liệt kê keyword + must-have | Ứng viên ghi SQL ở mức skill list. JD cần lưu kết quả interview vào database. | scenario | Hỏi ứng dụng SQL vào project, data model, bug query. | Verify practical SQL. | representative |
| A11 | Võ Tấn Trung | AI Interview Agent cho Tech Lead interview ứng viên Backend AI Engineer Intern | Agent có chọn đúng loại câu hỏi tiếp theo dựa trên quan hệ giữa CV và JD, đồng thời tạo được câu hỏi đủ cụ thể để đánh giá evidence, ownership, reasoning và JD fit không? | A-C06 | LLM API: CV/JD đều có + project rõ + cần scenario | CV có OpenAI API summarizer, JD cần backend tích hợp AI. Đã verify skill cơ bản, giờ cần work simulation. | scenario | Đưa scenario API nhận PDF, extract text, gọi LLM, lưu DB, hỏi flow. | JD work simulation. | high-risk |
| A12 | Võ Tấn Trung | AI Interview Agent cho Tech Lead interview ứng viên Backend AI Engineer Intern | Agent có chọn đúng loại câu hỏi tiếp theo dựa trên quan hệ giữa CV và JD, đồng thời tạo được câu hỏi đủ cụ thể để đánh giá evidence, ownership, reasoning và JD fit không? | A-C06 | LLM API: CV/JD đều có + project rõ + cần scenario | Tech Lead muốn biết ứng viên thiết kế flow nếu LLM timeout hoặc trả sai format trong backend AI. | high-risk | Hỏi design flow, retry, validation, fallback, logging. | Test applied reasoning. | high-risk |
| A13 | Võ Tấn Trung | AI Interview Agent cho Tech Lead interview ứng viên Backend AI Engineer Intern | Agent có chọn đúng loại câu hỏi tiếp theo dựa trên quan hệ giữa CV và JD, đồng thời tạo được câu hỏi đủ cụ thể để đánh giá evidence, ownership, reasoning và JD fit không? | A-C07 | CI/CD: chỉ JD có + CV không có + nice-to-have | JD có nice-to-have GitHub Actions, CV không nhắc. Must-have đã hỏi xong, cần câu scenario vừa sức. | scenario | Hỏi cách tiếp cận pipeline test/deploy đơn giản. | Test JD-only nice-to-have. | representative |
| A14 | Võ Tấn Trung | AI Interview Agent cho Tech Lead interview ứng viên Backend AI Engineer Intern | Agent có chọn đúng loại câu hỏi tiếp theo dựa trên quan hệ giữa CV và JD, đồng thời tạo được câu hỏi đủ cụ thể để đánh giá evidence, ownership, reasoning và JD fit không? | A-C07 | CI/CD: chỉ JD có + CV không có + nice-to-have | CV chưa có CI/CD, JD có tự động chạy test khi push. Cần đánh giá nền tảng của ứng viên. | gap-probe | Nếu chưa làm thì hỏi sẽ setup workflow như thế nào. | Learning approach. | representative |
| A15 | Võ Tấn Trung | AI Interview Agent cho Tech Lead interview ứng viên Backend AI Engineer Intern | Agent có chọn đúng loại câu hỏi tiếp theo dựa trên quan hệ giữa CV và JD, đồng thời tạo được câu hỏi đủ cụ thể để đánh giá evidence, ownership, reasoning và JD fit không? | A-C08 | Vector DB: chỉ CV có + keyword-only + cần follow-up | Ứng viên vừa nói “em có dùng vector database” nhưng không nói rõ trong project nào. | follow-up | Follow-up bằng chứng: DB nào, embedding schema, query, lỗi. | Test vague answer. | challenge |
| A16 | Võ Tấn Trung | AI Interview Agent cho Tech Lead interview ứng viên Backend AI Engineer Intern | Agent có chọn đúng loại câu hỏi tiếp theo dựa trên quan hệ giữa CV và JD, đồng thời tạo được câu hỏi đủ cụ thể để đánh giá evidence, ownership, reasoning và JD fit không? | A-C08 | Vector DB: chỉ CV có + keyword-only + cần follow-up | CV ghi Pinecone trong skill list, câu trả lời trước chỉ nói “em có tìm hiểu”. | ambiguous | Hỏi phần đã làm thật nếu có; nếu chưa thì chuyển, không lặp lại. | Avoid repeated probe. | challenge |
| A17 | Võ Tấn Trung | AI Interview Agent cho Tech Lead interview ứng viên Backend AI Engineer Intern | Agent có chọn đúng loại câu hỏi tiếp theo dựa trên quan hệ giữa CV và JD, đồng thời tạo được câu hỏi đủ cụ thể để đánh giá evidence, ownership, reasoning và JD fit không? | A-C09 | Python: CV/JD đều có + project rõ + cần follow-up | Ứng viên đã kể có viết Python service. Câu trả lời chung chung: “em xử lý bug bằng debug”. | follow-up | Hỏi bug cụ thể, cách reproduce, fix, trade-off. | Test follow-up depth. | representative |
| A18 | Võ Tấn Trung | AI Interview Agent cho Tech Lead interview ứng viên Backend AI Engineer Intern | Agent có chọn đúng loại câu hỏi tiếp theo dựa trên quan hệ giữa CV và JD, đồng thời tạo được câu hỏi đủ cụ thể để đánh giá evidence, ownership, reasoning và JD fit không? | A-C09 | Python: CV/JD đều có + project rõ + cần follow-up | CV có nhiều Python project, JD cần Python backend. Ứng viên nói “em làm logic chính” nhưng chưa rõ ownership. | follow-up | Hỏi module nào tự làm, test nào viết, lỗi nào gặp. | Verify ownership. | representative |
| A19 | Võ Tấn Trung | AI Interview Agent cho Tech Lead interview ứng viên Backend AI Engineer Intern | Agent có chọn đúng loại câu hỏi tiếp theo dựa trên quan hệ giữa CV và JD, đồng thời tạo được câu hỏi đủ cụ thể để đánh giá evidence, ownership, reasoning và JD fit không? | A-C10 | React: chỉ CV có + JD không cần + mở đầu | CV có React dashboard, JD backend AI intern. Chưa cover FastAPI, SQL, Docker. | priority-check | Không hỏi React trước; ưu tiên must-have backend/AI. | Test wrong priority. | challenge |
| A20 | Võ Tấn Trung | AI Interview Agent cho Tech Lead interview ứng viên Backend AI Engineer Intern | Agent có chọn đúng loại câu hỏi tiếp theo dựa trên quan hệ giữa CV và JD, đồng thời tạo được câu hỏi đủ cụ thể để đánh giá evidence, ownership, reasoning và JD fit không? | A-C10 | React: chỉ CV có + JD không cần + mở đầu | Agent cần chọn câu đầu. CV nói React rất nhiều, nhưng JD là backend AI với Python/FastAPI. | priority-check | Hỏi backend/AI must-have trước, React để sau nếu cần. | Avoid distraction. | challenge |

## 1.7. Coverage note cá nhân

Dataset cá nhân của Võ Tấn Trung cover tốt logic skill mapping CV/JD, đặc biệt là overlap và JD-only gap. Dataset chưa cover nhiều về soft-skill communication hoặc behavioral signal. Một số combination cả CV/JD đều không có được cố tình ít chọn vì không nên hỏi. High-risk nhất là Docker JD-only vì agent dễ hỏi yes/no hoặc giả định sai. Boundary case khó nhất là CV-only RAG vì có liên quan gần nhưng không phải must-have JD.

---

# 2. Phần cá nhân - Nguyễn Hoàng Thanh Tùng

## 2.1. Unit of AI Work

| Thành phần | Câu trả lời |
| :--- | :--- |
| Use case lựa chọn cho Day 21 | AI Interview Agent cho Prompt Automation Engineer Intern |
| Persona chính | Tech Lead/Automation Lead phỏng vấn ứng viên intern |
| Unit of AI Work | Sinh câu hỏi tiếp theo để đánh giá khả năng thiết kế workflow prompt/LLM automation |
| Input user đưa vào | CV, JD, skill overlap, câu trả lời trước, mục tiêu phần phỏng vấn |
| Output agent cần tạo | Câu hỏi Tech Lead về evidence, workflow scenario, error handling hoặc follow-up |
| Agent được phép làm gì? | Hỏi về automation flow, prompt versioning, output format validation, confidence handling |
| Agent không được phép làm gì? | Bắt code trực tiếp, hỏi quá level intern, hỏi prompt engineering chung chung không gắn JD |

## 2.2. Quality Question

| Câu hỏi | Câu trả lời |
| :--- | :--- |
| Quality question chính | Agent có tạo được câu hỏi Tech Lead gắn với JD Prompt Automation, vừa xác thực bằng chứng CV vừa đánh giá cách ứng viên xử lý workflow LLM thực tế không? |
| Vì sao quan trọng với user? | Vai trò này cần người biết biến prompt thành quy trình có validation, fallback và monitoring. |
| Nếu agent fail, hậu quả là gì? | Hỏi quá lý thuyết, không phát hiện ứng viên chưa từng xử lý output lỗi/confidence thấp. |
| Behavior bắt buộc | Hỏi workflow thực tế, format failure, confidence low, human review. |
| Behavior bị cấm | Chỉ hỏi “prompt là gì”, “bạn dùng ChatGPT chưa”, hoặc yêu cầu viết code như Technical Check. |

## 2.3. User Input Grid

| Dimension | Values | Vì sao làm agent phải đổi behavior? |
| :--- | :--- | :--- |
| `jd_task_type` | `email_classification`, `prompt_versioning`, `batch_processing`, `human_review` | Đổi task thì câu scenario và risk khác nhau. |
| `cv_evidence` | `workflow_project`, `prompt_keyword_only`, `no_prompt_evidence` | Quyết định xác thực claim hay hỏi gap. |
| `failure_risk` | `low`, `medium`, `high` | Quyết định độ sâu về fallback/escalation. |
| `candidate_answer_state` | `no_answer_yet`, `claims_experience`, `says_no_experience`, `vague_answer` | Quyết định follow-up hay chuyển scenario. |

## 2.4. Combinations cá nhân

| Combination ID | Dimension values | Expected behavior | Vì sao đáng test? | Loại |
| :---: | :--- | :--- | :--- | :--- |
| B-C01 | email classification + workflow project + high + no answer | Hỏi workflow classify email, confidence, routing, wrong format | Core JD scenario | high-risk |
| B-C02 | prompt versioning + prompt keyword only + medium + no answer | Hỏi bằng chứng quản lý prompt version/eval | Verify keyword | challenge |
| B-C03 | batch processing + no prompt evidence + medium + no answer | Hỏi cách tiếp cận batch nếu chưa có kinh nghiệm | JD-only gap | representative |
| B-C04 | human review + workflow project + high + claims experience | Follow-up ownership và escalation rule | Verify implementation | high-risk |
| B-C05 | email classification + prompt keyword only + high + vague answer | Follow-up format, confidence, examples, metrics | Vague answer | challenge |
| B-C06 | prompt versioning + no prompt evidence + low + says no experience | Chuyển sang scenario intern-level, không hỏi lặp | Avoid repeated gap | representative |
| B-C07 | batch processing + workflow project + high + claims experience | Hỏi retry, rate limit, partial failure, logs | Work simulation | high-risk |
| B-C08 | human review + no prompt evidence + medium + no answer | Hỏi thiết kế handoff khi confidence thấp | JD scenario | representative |
| B-C09 | email classification + no prompt evidence + high + says no experience | Hỏi cách phân tích problem và flow, đánh dấu no direct evidence | Gap handling | challenge |
| B-C10 | prompt versioning + workflow project + medium + vague answer | Hỏi cách so sánh prompt mới/cũ và rollback | Trade-off | challenge |

## 2.5. Prompt đã dùng để generate inputs

```text
Bạn là người thiết kế test input cho AI Interview Agent.

Use case: AI Interview Agent hỗ trợ Tech Lead phỏng vấn Prompt Automation Engineer Intern dựa trên CV, JD và câu trả lời trước đó.
Quality question: Agent có tạo được câu hỏi Tech Lead gắn với JD Prompt Automation, vừa xác thực bằng chứng CV vừa đánh giá cách ứng viên xử lý workflow LLM thực tế không?

Combinations: Dùng đúng 10 combinations B-C01 đến B-C10 trong bảng Combinations cá nhân, gồm các lát cắt về email classification, prompt versioning, batch processing, human review, mức evidence trong CV, failure risk và trạng thái câu trả lời của ứng viên.

Hãy viết mỗi combination thành 2 input tự nhiên. Mỗi input gồm CV summary, JD summary, interview stage hoặc candidate_answer_state và nếu cần có câu trả lời gần nhất của ứng viên.
Không tự thêm combination mới.
Không thay đổi intent, risk hoặc context của combination gốc.
Output dạng bảng gồm `combination_id`, `user_input`, `style`, `notes`.
```

**Human filtering note:** AI chỉ được dùng để paraphrase hoặc viết input tự nhiên. Người học giữ quyền quyết định coverage. Các input generic, sai intent, tự thêm context, làm mất ambiguity hoặc trùng nhau đã bị loại bỏ. Scenario Dataset v0 bên dưới là phiên bản đã lọc cuối cùng.

## 2.6. Scenario Dataset v0 - Nguyễn Hoàng Thanh Tùng

| scenario_id | owner | use_case | quality_question | combination_id | dimension_values | user_input | style | expected_behavior | why_included | set_type |
| :---: | :--- | :--- | :--- | :---: | :--- | :--- | :--- | :--- | :--- | :--- |
| B01 | Nguyễn Hoàng Thanh Tùng | AI Interview Agent cho Prompt Automation Engineer Intern | Agent có tạo được câu hỏi Tech Lead gắn với JD Prompt Automation, vừa xác thực bằng chứng CV vừa đánh giá cách ứng viên xử lý workflow LLM thực tế không? | B-C01 | email classification + workflow project + high + no answer | CV có workflow phân loại feedback bằng LLM. JD cần classify email khách hàng và route ticket. | scenario | Hỏi thiết kế flow, confidence, route, wrong format. | Core JD scenario. | high-risk |
| B02 | Nguyễn Hoàng Thanh Tùng | AI Interview Agent cho Prompt Automation Engineer Intern | Agent có tạo được câu hỏi Tech Lead gắn với JD Prompt Automation, vừa xác thực bằng chứng CV vừa đánh giá cách ứng viên xử lý workflow LLM thực tế không? | B-C01 | email classification + workflow project + high + no answer | JD: batch email customer -> LLM intent -> confidence -> assign team. CV có automation bot. | scenario | Tạo work simulation gắn JD, hỏi error handling. | Test applied reasoning. | high-risk |
| B03 | Nguyễn Hoàng Thanh Tùng | AI Interview Agent cho Prompt Automation Engineer Intern | Agent có tạo được câu hỏi Tech Lead gắn với JD Prompt Automation, vừa xác thực bằng chứng CV vừa đánh giá cách ứng viên xử lý workflow LLM thực tế không? | B-C02 | prompt versioning + prompt keyword only + medium + no answer | CV chỉ ghi “prompt engineering”. JD cần maintain prompt templates và version. | gap-probe | Hỏi bằng chứng về versioning/eval prompt, không hỏi lý thuyết. | Verify keyword. | challenge |
| B04 | Nguyễn Hoàng Thanh Tùng | AI Interview Agent cho Prompt Automation Engineer Intern | Agent có tạo được câu hỏi Tech Lead gắn với JD Prompt Automation, vừa xác thực bằng chứng CV vừa đánh giá cách ứng viên xử lý workflow LLM thực tế không? | B-C02 | prompt versioning + prompt keyword only + medium + no answer | Ứng viên liệt kê prompt engineering nhưng không nói project. Tech Lead muốn biết có từng thay đổi prompt có kiểm soát chưa. | context-rich | Hỏi cách track version, compare output, rollback. | Practical prompt ops. | challenge |
| B05 | Nguyễn Hoàng Thanh Tùng | AI Interview Agent cho Prompt Automation Engineer Intern | Agent có tạo được câu hỏi Tech Lead gắn với JD Prompt Automation, vừa xác thực bằng chứng CV vừa đánh giá cách ứng viên xử lý workflow LLM thực tế không? | B-C03 | batch processing + no prompt evidence + medium + no answer | JD cần xử lý batch 1000 email, CV không có automation/prompt project. | high-risk | Hỏi batch experience; nếu chưa có thì hỏi chia batch, retry, log. | Gap probe. | representative |
| B06 | Nguyễn Hoàng Thanh Tùng | AI Interview Agent cho Prompt Automation Engineer Intern | Agent có tạo được câu hỏi Tech Lead gắn với JD Prompt Automation, vừa xác thực bằng chứng CV vừa đánh giá cách ứng viên xử lý workflow LLM thực tế không? | B-C03 | batch processing + no prompt evidence + medium + no answer | CV mainly data cleaning, JD có LLM automation batch. Cần câu hỏi vừa sức intern. | gap-probe | Hỏi adjacent experience và approach. | Learning signal. | representative |
| B07 | Nguyễn Hoàng Thanh Tùng | AI Interview Agent cho Prompt Automation Engineer Intern | Agent có tạo được câu hỏi Tech Lead gắn với JD Prompt Automation, vừa xác thực bằng chứng CV vừa đánh giá cách ứng viên xử lý workflow LLM thực tế không? | B-C04 | human review + workflow project + high + claims experience | Ứng viên nói đã làm bot tự động route ticket. Cần follow-up về lúc nào chuyển human. | follow-up | Hỏi rule escalation, ownership, case sai route. | Verify workflow. | high-risk |
| B08 | Nguyễn Hoàng Thanh Tùng | AI Interview Agent cho Prompt Automation Engineer Intern | Agent có tạo được câu hỏi Tech Lead gắn với JD Prompt Automation, vừa xác thực bằng chứng CV vừa đánh giá cách ứng viên xử lý workflow LLM thực tế không? | B-C04 | human review + workflow project + high + claims experience | Câu trả lời trước: “nếu confidence thấp thì gửi người xem”. Cần đào sâu. | follow-up | Hỏi threshold, queue, metadata, audit trail. | Test depth. | high-risk |
| B09 | Nguyễn Hoàng Thanh Tùng | AI Interview Agent cho Prompt Automation Engineer Intern | Agent có tạo được câu hỏi Tech Lead gắn với JD Prompt Automation, vừa xác thực bằng chứng CV vừa đánh giá cách ứng viên xử lý workflow LLM thực tế không? | B-C05 | email classification + prompt keyword only + high + vague answer | Ứng viên trả lời “em prompt cho model trả JSON” nhưng không nói nếu model sai format. | ambiguous | Hỏi validation, retry, fallback, examples. | Wrong format risk. | challenge |
| B10 | Nguyễn Hoàng Thanh Tùng | AI Interview Agent cho Prompt Automation Engineer Intern | Agent có tạo được câu hỏi Tech Lead gắn với JD Prompt Automation, vừa xác thực bằng chứng CV vừa đánh giá cách ứng viên xử lý workflow LLM thực tế không? | B-C05 | email classification + prompt keyword only + high + vague answer | CV chỉ ghi prompt. Khi hỏi classification, ứng viên nói “em sẽ viết prompt tốt hơn”. | ambiguous | Đào sâu metrics, confidence, format guardrails. | Avoid generic. | challenge |
| B11 | Nguyễn Hoàng Thanh Tùng | AI Interview Agent cho Prompt Automation Engineer Intern | Agent có tạo được câu hỏi Tech Lead gắn với JD Prompt Automation, vừa xác thực bằng chứng CV vừa đánh giá cách ứng viên xử lý workflow LLM thực tế không? | B-C06 | prompt versioning + no prompt evidence + low + says no experience | Ứng viên nói chưa từng quản lý prompt version. JD nice-to-have prompt templates. | gap-probe | Không hỏi lặp; chuyển sang scenario intern-level. | Avoid repeated gap. | representative |
| B12 | Nguyễn Hoàng Thanh Tùng | AI Interview Agent cho Prompt Automation Engineer Intern | Agent có tạo được câu hỏi Tech Lead gắn với JD Prompt Automation, vừa xác thực bằng chứng CV vừa đánh giá cách ứng viên xử lý workflow LLM thực tế không? | B-C06 | prompt versioning + no prompt evidence + low + says no experience | Candidate nói mới dùng prompt cá nhân, chưa làm team workflow. | gap-probe | Hỏi intern-level approach và đánh dấu no direct evidence. | Gap handling. | representative |
| B13 | Nguyễn Hoàng Thanh Tùng | AI Interview Agent cho Prompt Automation Engineer Intern | Agent có tạo được câu hỏi Tech Lead gắn với JD Prompt Automation, vừa xác thực bằng chứng CV vừa đánh giá cách ứng viên xử lý workflow LLM thực tế không? | B-C07 | batch processing + workflow project + high + claims experience | Ứng viên claim từng chạy LLM batch. JD cần batch email mỗi ngày. | follow-up | Hỏi rate limit, retry, partial failure, log. | Verify production-ish thinking. | high-risk |
| B14 | Nguyễn Hoàng Thanh Tùng | AI Interview Agent cho Prompt Automation Engineer Intern | Agent có tạo được câu hỏi Tech Lead gắn với JD Prompt Automation, vừa xác thực bằng chứng CV vừa đánh giá cách ứng viên xử lý workflow LLM thực tế không? | B-C07 | batch processing + workflow project + high + claims experience | Candidate nói đã làm batch summarize reviews. Cần câu follow-up khó hơn. | high-risk | Hỏi lỗi giữa batch, idempotency, cost. | High-risk batch. | high-risk |
| B15 | Nguyễn Hoàng Thanh Tùng | AI Interview Agent cho Prompt Automation Engineer Intern | Agent có tạo được câu hỏi Tech Lead gắn với JD Prompt Automation, vừa xác thực bằng chứng CV vừa đánh giá cách ứng viên xử lý workflow LLM thực tế không? | B-C08 | human review + no prompt evidence + medium + no answer | JD cần route ticket sang human nếu low confidence; CV không có AI automation. | scenario | Hỏi thiết kế flow human-in-the-loop. | JD work simulation. | representative |
| B16 | Nguyễn Hoàng Thanh Tùng | AI Interview Agent cho Prompt Automation Engineer Intern | Agent có tạo được câu hỏi Tech Lead gắn với JD Prompt Automation, vừa xác thực bằng chứng CV vừa đánh giá cách ứng viên xử lý workflow LLM thực tế không? | B-C08 | human review + no prompt evidence + medium + no answer | CV có customer support part-time nhưng không có AI automation. JD cần human review. | context-rich | Hỏi cách kết hợp AI suggestion và người duyệt. | Adjacent experience. | representative |
| B17 | Nguyễn Hoàng Thanh Tùng | AI Interview Agent cho Prompt Automation Engineer Intern | Agent có tạo được câu hỏi Tech Lead gắn với JD Prompt Automation, vừa xác thực bằng chứng CV vừa đánh giá cách ứng viên xử lý workflow LLM thực tế không? | B-C09 | email classification + no prompt evidence + high + says no experience | Ứng viên nói chưa từng dùng LLM classify email. JD coi đây là task chính. | high-risk | Hỏi scenario phân tích flow, ghi nhận chưa có direct evidence. | Must-have gap. | challenge |
| B18 | Nguyễn Hoàng Thanh Tùng | AI Interview Agent cho Prompt Automation Engineer Intern | Agent có tạo được câu hỏi Tech Lead gắn với JD Prompt Automation, vừa xác thực bằng chứng CV vừa đánh giá cách ứng viên xử lý workflow LLM thực tế không? | B-C09 | email classification + no prompt evidence + high + says no experience | Candidate thành thật nói “em mới dùng ChatGPT hỏi đáp”. Cần câu tiếp theo. | gap-probe | Chuyển qua work simulation có hướng dẫn, không ép claim. | Evaluate reasoning. | challenge |
| B19 | Nguyễn Hoàng Thanh Tùng | AI Interview Agent cho Prompt Automation Engineer Intern | Agent có tạo được câu hỏi Tech Lead gắn với JD Prompt Automation, vừa xác thực bằng chứng CV vừa đánh giá cách ứng viên xử lý workflow LLM thực tế không? | B-C10 | prompt versioning + workflow project + medium + vague answer | Candidate nói “em update prompt khi thấy sai” nhưng không nói cách biết prompt mới tốt hơn. | follow-up | Hỏi eval set, compare, rollback. | Test trade-off. | challenge |
| B20 | Nguyễn Hoàng Thanh Tùng | AI Interview Agent cho Prompt Automation Engineer Intern | Agent có tạo được câu hỏi Tech Lead gắn với JD Prompt Automation, vừa xác thực bằng chứng CV vừa đánh giá cách ứng viên xử lý workflow LLM thực tế không? | B-C10 | prompt versioning + workflow project + medium + vague answer | CV có chatbot prompt project; câu trả lời trước chưa nói cách quản lý thay đổi prompt. | context-rich | Hỏi quy trình change control và metric. | Prompt ops depth. | challenge |

## 2.7. Coverage note cá nhân

Dataset B cover tốt Prompt Automation, đặc biệt workflow, confidence, wrong format, human review và batch failure. Chưa cover sâu về security/privacy của email khách hàng. Một số skill code-level được bỏ qua vì Tech Lead interview không phải Technical Check. High-risk nhất là email classification sai route. Boundary case khó là ứng viên không có prompt evidence nhưng có tư duy workflow tốt.

---

# 3. Phần cá nhân - Nguyễn Minh Đức

## 3.1. Unit of AI Work

| Thành phần | Câu trả lời |
| :--- | :--- |
| Use case lựa chọn cho Day 21 | AI Interview Agent cho Data/AI Backend Intern |
| Persona chính | Tech Lead phỏng vấn ứng viên intern cần làm data pipeline và AI service |
| Unit of AI Work | Sinh câu hỏi tiếp theo dựa trên CV/JD/câu trả lời để đánh giá data handling, API design, debugging |
| Input user đưa vào | CV, JD, skill relation, câu trả lời gần nhất, mức ưu tiên JD |
| Output agent cần tạo | Câu hỏi xác thực evidence hoặc JD scenario về data/API/error handling |
| Agent được phép làm gì? | Hỏi project cụ thể, data flow, schema, edge case, debug, trade-off |
| Agent không được phép làm gì? | Hỏi skill ngoài JD, hỏi quá khó, chuyển thành coding challenge |

## 3.2. Quality Question

| Câu hỏi | Câu trả lời |
| :--- | :--- |
| Quality question chính | Agent có phân biệt được khi nào cần hỏi evidence về data/API trong CV và khi nào cần đưa JD scenario để đánh giá flow xử lý dữ liệu không? |
| Vì sao quan trọng với user? | Vai trò AI backend thường fail ở data quality, async job, error handling, không chỉ ở việc gọi model. |
| Nếu agent fail, hậu quả là gì? | Bỏ sót ứng viên không hiểu data flow, hoặc hỏi quá chung chung không có tín hiệu. |
| Behavior bắt buộc | Hỏi flow input-output, validation, DB, async/background job, failure mode. |
| Behavior bị cấm | Chỉ hỏi tool keyword, bắt viết code, hỏi ML theory không liên quan JD. |

## 3.3. User Input Grid

| Dimension | Values | Vì sao làm agent phải đổi behavior? |
| :--- | :--- | :--- |
| `work_area` | `api_design`, `data_pipeline`, `model_integration`, `debugging` | Mỗi area cần câu hỏi và expected behavior khác. |
| `skill_relation` | `overlap`, `jd_only`, `cv_only` | Quyết định verify/gap/secondary signal. |
| `context_quality` | `enough_context`, `missing_context`, `conflicting_claim` | Quyết định hỏi làm rõ hay đào sâu. |
| `risk_type` | `wrong_action`, `data_loss`, `poor_user_experience`, `low_risk` | Quyết định mức ưu tiên và độ sâu follow-up. |

## 3.4. Combinations cá nhân

| Combination ID | Dimension values | Expected behavior | Vì sao đáng test? | Loại |
| :---: | :--- | :--- | :--- | :--- |
| C-C01 | api design + overlap + enough context + wrong action | Verify API project, endpoint, validation, auth/error | Core API evidence | representative |
| C-C02 | data pipeline + jd only + missing context + data loss | Gap probe và scenario ETL/data cleaning | JD-only high risk | high-risk |
| C-C03 | model integration + overlap + enough context + poor UX | Scenario gọi LLM, latency, timeout, fallback | AI service flow | high-risk |
| C-C04 | debugging + overlap + conflicting claim + wrong action | Follow-up làm rõ claim mâu thuẫn | Detect inconsistency | challenge |
| C-C05 | api design + cv only + enough context + low risk | Hỏi sau must-have nếu liên quan | Secondary signal | representative |
| C-C06 | data pipeline + overlap + missing context + data loss | Hỏi data validation và ownership | Missing info | high-risk |
| C-C07 | model integration + jd only + missing context + poor UX | Hỏi cách tiếp cận LLM integration nếu chưa có | JD gap scenario | challenge |
| C-C08 | debugging + jd only + missing context + wrong action | Hỏi scenario debug production issue ở mức intern | Work simulation | challenge |
| C-C09 | api design + overlap + conflicting claim + wrong action | Hỏi để reconcile CV claim và answer | Inconsistency | challenge |
| C-C10 | data pipeline + cv only + enough context + low risk | Chọn hỏi nếu bổ sung fit, không trước must-have | Priority check | representative |

## 3.5. Prompt đã dùng để generate inputs

```text
Bạn là người thiết kế test input cho AI Interview Agent.

Use case: AI Interview Agent hỗ trợ Tech Lead phỏng vấn Data/AI Backend Intern dựa trên CV, JD và câu trả lời trước đó.
Quality question: Agent có phân biệt được khi nào cần hỏi evidence về data/API trong CV và khi nào cần đưa JD scenario để đánh giá flow xử lý dữ liệu không?

Combinations: Dùng đúng 10 combinations C-C01 đến C-C10 trong bảng Combinations cá nhân, gồm các lát cắt về api design, data pipeline, model integration, debugging, skill relation, context quality và risk type.

Hãy viết mỗi combination thành 2 input tự nhiên. Mỗi input gồm CV summary, JD summary, interview stage hoặc context_quality và nếu cần có câu trả lời gần nhất của ứng viên.
Không tự thêm combination mới.
Không thay đổi intent, risk hoặc context của combination gốc.
Output dạng bảng gồm `combination_id`, `user_input`, `style`, `notes`.
```

**Human filtering note:** AI chỉ được dùng để paraphrase hoặc viết input tự nhiên. Người học giữ quyền quyết định coverage. Các input generic, sai intent, tự thêm context, làm mất ambiguity hoặc trùng nhau đã bị loại bỏ. Scenario Dataset v0 bên dưới là phiên bản đã lọc cuối cùng.

## 3.6. Scenario Dataset v0 - Nguyễn Minh Đức

| scenario_id | owner | use_case | quality_question | combination_id | dimension_values | user_input | style | expected_behavior | why_included | set_type |
| :---: | :--- | :--- | :--- | :---: | :--- | :--- | :--- | :--- | :--- | :--- |
| C01 | Nguyễn Minh Đức | AI Interview Agent cho Data/AI Backend Intern | Agent có phân biệt được khi nào cần hỏi evidence về data/API trong CV và khi nào cần đưa JD scenario để đánh giá flow xử lý dữ liệu không? | C-C01 | api design + overlap + enough context + wrong action | CV có REST API project, JD cần API cho AI feature. Cần câu hỏi đầu về API. | concise | Hỏi endpoint, validation, error handling, ownership. | Core API evidence. | representative |
| C02 | Nguyễn Minh Đức | AI Interview Agent cho Data/AI Backend Intern | Agent có phân biệt được khi nào cần hỏi evidence về data/API trong CV và khi nào cần đưa JD scenario để đánh giá flow xử lý dữ liệu không? | C-C01 | api design + overlap + enough context + wrong action | Ứng viên ghi “designed backend APIs” và JD cần nhận input từ frontend gửi sang AI service. | context-rich | Verify API flow, edge cases, bug. | Avoid generic API question. | representative |
| C03 | Nguyễn Minh Đức | AI Interview Agent cho Data/AI Backend Intern | Agent có phân biệt được khi nào cần hỏi evidence về data/API trong CV và khi nào cần đưa JD scenario để đánh giá flow xử lý dữ liệu không? | C-C02 | data pipeline + jd only + missing context + data loss | JD cần clean/import CSV ứng viên, CV không có data pipeline. | high-risk | Hỏi adjacent experience; nếu chưa có thì scenario validate CSV, missing fields. | Data loss risk. | high-risk |
| C04 | Nguyễn Minh Đức | AI Interview Agent cho Data/AI Backend Intern | Agent có phân biệt được khi nào cần hỏi evidence về data/API trong CV và khi nào cần đưa JD scenario để đánh giá flow xử lý dữ liệu không? | C-C02 | data pipeline + jd only + missing context + data loss | CV có backend, không có ETL. JD nói cần preprocess resume data trước khi gọi AI. | scenario | Hỏi cách xử lý dữ liệu thiếu/sai format/idempotency. | Test data handling. | high-risk |
| C05 | Nguyễn Minh Đức | AI Interview Agent cho Data/AI Backend Intern | Agent có phân biệt được khi nào cần hỏi evidence về data/API trong CV và khi nào cần đưa JD scenario để đánh giá flow xử lý dữ liệu không? | C-C03 | model integration + overlap + enough context + poor UX | CV có LLM summarizer, JD cần gọi model trong backend. | scenario | Hỏi latency, timeout, retry, fallback, user message. | AI service flow. | high-risk |
| C06 | Nguyễn Minh Đức | AI Interview Agent cho Data/AI Backend Intern | Agent có phân biệt được khi nào cần hỏi evidence về data/API trong CV và khi nào cần đưa JD scenario để đánh giá flow xử lý dữ liệu không? | C-C03 | model integration + overlap + enough context + poor UX | Candidate đã làm chatbot API. Tech Lead cần biết nếu model chậm/lỗi thì flow ra sao. | high-risk | Hỏi async/background job, status, error handling. | Poor UX risk. | high-risk |
| C07 | Nguyễn Minh Đức | AI Interview Agent cho Data/AI Backend Intern | Agent có phân biệt được khi nào cần hỏi evidence về data/API trong CV và khi nào cần đưa JD scenario để đánh giá flow xử lý dữ liệu không? | C-C04 | debugging + overlap + conflicting claim + wrong action | CV nói “fixed backend bugs”, nhưng ứng viên vừa nói chưa từng debug API lỗi thật. | conflicting-claim | Hỏi làm rõ claim bằng ví dụ bug cụ thể nếu có. | Detect inconsistency. | challenge |
| C08 | Nguyễn Minh Đức | AI Interview Agent cho Data/AI Backend Intern | Agent có phân biệt được khi nào cần hỏi evidence về data/API trong CV và khi nào cần đưa JD scenario để đánh giá flow xử lý dữ liệu không? | C-C04 | debugging + overlap + conflicting claim + wrong action | Candidate ghi debugging trong CV, câu trả lời lại chỉ nói làm theo tutorial. | conflicting-claim | Ask ownership/reproduce/fix, không accuse. | Verify evidence. | challenge |
| C09 | Nguyễn Minh Đức | AI Interview Agent cho Data/AI Backend Intern | Agent có phân biệt được khi nào cần hỏi evidence về data/API trong CV và khi nào cần đưa JD scenario để đánh giá flow xử lý dữ liệu không? | C-C05 | api design + cv only + enough context + low risk | CV có GraphQL API, JD chủ yếu data pipeline và LLM. Must-have đã hỏi xong. | priority-check | Hỏi API như signal bổ sung, không ưu tiên trước. | Priority. | representative |
| C10 | Nguyễn Minh Đức | AI Interview Agent cho Data/AI Backend Intern | Agent có phân biệt được khi nào cần hỏi evidence về data/API trong CV và khi nào cần đưa JD scenario để đánh giá flow xử lý dữ liệu không? | C-C05 | api design + cv only + enough context + low risk | Sau khi cover data pipeline, Tech Lead muốn hỏi thêm GraphQL project trong CV. | follow-up | Verify related backend ownership. | Secondary signal. | representative |
| C11 | Nguyễn Minh Đức | AI Interview Agent cho Data/AI Backend Intern | Agent có phân biệt được khi nào cần hỏi evidence về data/API trong CV và khi nào cần đưa JD scenario để đánh giá flow xử lý dữ liệu không? | C-C06 | data pipeline + overlap + missing context + data loss | CV ghi “processed resume dataset” nhưng không nói schema/quality checks; JD cần reliable data import. | gap-probe | Hỏi data validation, duplicates, bad rows, ownership. | Data quality. | high-risk |
| C12 | Nguyễn Minh Đức | AI Interview Agent cho Data/AI Backend Intern | Agent có phân biệt được khi nào cần hỏi evidence về data/API trong CV và khi nào cần đưa JD scenario để đánh giá flow xử lý dữ liệu không? | C-C06 | data pipeline + overlap + missing context + data loss | Ứng viên có data cleaning project nhưng CV quá ngắn. Cần câu hỏi đào sâu. | context-rich | Hỏi pipeline steps và lỗi đã gặp. | Verify data evidence. | high-risk |
| C13 | Nguyễn Minh Đức | AI Interview Agent cho Data/AI Backend Intern | Agent có phân biệt được khi nào cần hỏi evidence về data/API trong CV và khi nào cần đưa JD scenario để đánh giá flow xử lý dữ liệu không? | C-C07 | model integration + jd only + missing context + poor UX | JD cần tích hợp LLM API, CV không có AI model integration. | gap-probe | Hỏi đã từng gọi external API chưa; nếu chưa thì thiết kế flow LLM. | JD gap scenario. | challenge |
| C14 | Nguyễn Minh Đức | AI Interview Agent cho Data/AI Backend Intern | Agent có phân biệt được khi nào cần hỏi evidence về data/API trong CV và khi nào cần đưa JD scenario để đánh giá flow xử lý dữ liệu không? | C-C07 | model integration + jd only + missing context + poor UX | CV backend CRUD, JD AI backend. Cần câu hỏi không làm ứng viên phải nói có. | scenario | Hỏi adjacent API integration và cách học/tiếp cận. | Learning ability. | challenge |
| C15 | Nguyễn Minh Đức | AI Interview Agent cho Data/AI Backend Intern | Agent có phân biệt được khi nào cần hỏi evidence về data/API trong CV và khi nào cần đưa JD scenario để đánh giá flow xử lý dữ liệu không? | C-C08 | debugging + jd only + missing context + wrong action | JD cần debug API/model failures, CV không có debugging evidence. | scenario | Hỏi scenario request fail giữa pipeline, cách isolate lỗi. | Work simulation. | challenge |
| C16 | Nguyễn Minh Đức | AI Interview Agent cho Data/AI Backend Intern | Agent có phân biệt được khi nào cần hỏi evidence về data/API trong CV và khi nào cần đưa JD scenario để đánh giá flow xử lý dữ liệu không? | C-C08 | debugging + jd only + missing context + wrong action | Tech Lead muốn test debugging reasoning khi CV không nói debug. | gap-probe | Hỏi steps reproduce/logs/check boundary. | Debug reasoning. | challenge |
| C17 | Nguyễn Minh Đức | AI Interview Agent cho Data/AI Backend Intern | Agent có phân biệt được khi nào cần hỏi evidence về data/API trong CV và khi nào cần đưa JD scenario để đánh giá flow xử lý dữ liệu không? | C-C09 | api design + overlap + conflicting claim + wrong action | CV nói tự thiết kế API, nhưng ứng viên nói “team em có sẵn endpoint”. | conflicting-claim | Hỏi phần nào ứng viên tự làm, decision nào tự chọn. | Ownership conflict. | challenge |
| C18 | Nguyễn Minh Đức | AI Interview Agent cho Data/AI Backend Intern | Agent có phân biệt được khi nào cần hỏi evidence về data/API trong CV và khi nào cần đưa JD scenario để đánh giá flow xử lý dữ liệu không? | C-C09 | api design + overlap + conflicting claim + wrong action | Candidate claim API ownership trong CV, answer lại cho thấy chỉ consume API. | conflicting-claim | Clarify ownership without over-accusing. | Evidence quality. | challenge |
| C19 | Nguyễn Minh Đức | AI Interview Agent cho Data/AI Backend Intern | Agent có phân biệt được khi nào cần hỏi evidence về data/API trong CV và khi nào cần đưa JD scenario để đánh giá flow xử lý dữ liệu không? | C-C10 | data pipeline + cv only + enough context + low risk | CV có data visualization pipeline, JD không cần nhiều. Must-have AI integration chưa hỏi. | priority-check | Không hỏi data viz trước must-have. | Avoid wrong priority. | representative |
| C20 | Nguyễn Minh Đức | AI Interview Agent cho Data/AI Backend Intern | Agent có phân biệt được khi nào cần hỏi evidence về data/API trong CV và khi nào cần đưa JD scenario để đánh giá flow xử lý dữ liệu không? | C-C10 | data pipeline + cv only + enough context + low risk | Agent phải chọn câu hỏi tiếp theo, CV có ETL hobby project nhưng JD ưu tiên API/LLM. | priority-check | Để sau hoặc liên hệ nếu cần data handling. | Priority check. | representative |

## 3.7. Coverage note cá nhân

Dataset C cover tốt API/data/model integration/debugging trong Tech Lead interview. Chưa cover sâu về security, privacy và cost control của AI service. Một số câu hỏi về ML theory không được chọn vì không phù hợp JD intern backend. High-risk nhất là data pipeline missing context vì có thể gây data loss. Boundary case khó là conflicting claim giữa CV và câu trả lời.

---

# 4. Group Merge

## 4.1. Bảng gom dimensions của các thành viên

| Thành viên | Dimensions ban đầu | Chuẩn hóa thành |
| :--- | :--- | :--- |
| Võ Tấn Trung | skill_overlap_type, evidence_level_in_cv, interview_stage, skill_priority | skill_relation, evidence_strength, interview_state, jd_priority |
| Nguyễn Hoàng Thanh Tùng | jd_task_type, cv_evidence, failure_risk, candidate_answer_state | jd_work_area, evidence_strength, risk_level, interview_state |
| Nguyễn Minh Đức | work_area, skill_relation, context_quality, risk_type | jd_work_area, skill_relation, context_quality, risk_level |

## 4.2. Dimensions chuẩn hóa cho Scenario Dataset v1

| Dimension | Values chuẩn hóa | Lý do chọn |
| :--- | :--- | :--- |
| `skill_relation` | `cv_jd_overlap`, `jd_only`, `cv_only`, `neither` | Cốt lõi của logic hỏi: verify, gap probe, secondary, skip. |
| `evidence_strength` | `clear_evidence`, `keyword_only`, `absent`, `conflicting` | Cho biết câu hỏi cần đào sâu mức nào. |
| `jd_work_area` | `backend_api`, `docker_deploy`, `llm_integration`, `prompt_automation`, `data_pipeline`, `debugging`, `human_review` | Gắn câu hỏi với yêu cầu công việc thật trong JD. |
| `interview_state` | `opening`, `after_must_have`, `scenario_needed`, `follow_up_needed`, `says_no_experience` | Quyết định câu hỏi mới, follow-up hay chuyển scenario. |
| `risk_level` | `low`, `medium`, `high` | Ưu tiên high-risk và failure cost cao. |

## 4.3. Merge/Dedup Decisions

| Source rows | Decision | Lý do |
| :--- | :--- | :--- |
| A01/A02 | merged | Cùng test FastAPI overlap clear evidence, giữ wording rõ hơn về endpoint/bug. |
| A03/A04 | merged | Cùng test Docker JD-only, giữ bản có “nếu chưa có thì tiếp cận”. |
| B01/B02 | merged | Cùng JD email classification scenario, giữ bản có confidence/wrong format/routing. |
| B09/B10 | merged | Cùng vague prompt answer, giữ bản hỏi validation/metrics. |
| C03/C04 | merged | Cùng data pipeline JD-only, kết hợp missing fields/idempotency. |
| C07/C08 | merged | Cùng conflicting debugging claim, giữ follow-up không accuse. |
| A07/A08 | kept one | Negative case neither skill, chỉ cần một row để test skip irrelevant. |
| A19/A20 | kept one | Negative priority React cv-only unrelated, chỉ cần một row. |
| B11/B12 | merged | Says no experience prompt versioning, giữ scenario intern-level. |
| C17/C18 | merged | API ownership conflict, giữ câu hỏi làm rõ phần tự làm. |

---

# 5. Coverage Matrix

| Slice / value | Số rows hiện có | Đủ chưa? | Ghi chú |
| :--- | :---: | :---: | :--- |
| cv_jd_overlap | 11 | Đủ | Có FastAPI, SQL, LLM API, Python, prompt workflow, API/data/debug. |
| jd_only | 9 | Đủ | Có Docker, CI/CD, batch, human review, data pipeline, LLM integration. |
| cv_only | 5 | Đủ vừa | Có RAG, Vector DB, GraphQL/API, React negative, data viz. |
| neither | 1 | Đủ cho negative case | Chỉ cần ít vì expected là không hỏi. |
| clear_evidence | 9 | Đủ | Tập trung verify ownership. |
| keyword_only | 6 | Đủ | Test hỏi claim mơ hồ. |
| absent | 8 | Đủ | Test JD gap không giả định. |
| conflicting | 3 | Đủ vừa | Cần follow-up không accuse. |
| JD work simulation | 10 | Đủ | Có backend AI, prompt automation, data pipeline, debug. |
| follow-up needed | 8 | Đủ | Có vague answer, claims experience, conflict. |
| high risk | 12 | Đủ | Docker, LLM, email routing, data loss, wrong action. |
| happy path representative | 10 | Không over-sample | Có nhưng không áp đảo challenge/high-risk. |

---

# 6. Scenario Dataset v1 - Final 30 Rows

| scenario_id | source_owner | use_case | dimension_values | user_input | expected_behavior | risk_if_fail | why_included | set_type | merge_decision |
| :---: | :--- | :--- | :--- | :--- | :--- | :--- | :--- | :--- | :--- |
| G01 | Võ Tấn Trung | Tech Lead Interview Agent | skill_relation=cv_jd_overlap; evidence=clear_evidence; work_area=backend_api; state=opening; risk=medium | CV có project “AI Resume API dùng FastAPI, PostgreSQL”; JD cần FastAPI backend. Mới bắt đầu phỏng vấn, cần câu hỏi đầu về backend. | Hỏi bằng chứng FastAPI: project nào, endpoint nào tự làm, validation/error handling, bug khó nhất. | Nếu hỏi generic sẽ không verify được claim must-have. | Core overlap skill verification. | representative | merged |
| G02 | Võ Tấn Trung | Tech Lead Interview Agent | skill_relation=jd_only; evidence=absent; work_area=docker_deploy; state=opening; risk=high | JD bắt buộc Docker để deploy backend, nhưng CV không có Docker. Cần hỏi mà không làm ứng viên bị “yes/no”. | Hỏi đã từng containerize/deploy backend chưa; nếu có thì mô tả một lần tự làm; nếu chưa thì tiếp cận đóng gói FastAPI app. | Agent có thể giả định sai kinh nghiệm hoặc bỏ sót must-have JD. | JD-only must-have gap. | high-risk | merged |
| G03 | Võ Tấn Trung | Tech Lead Interview Agent | skill_relation=cv_only; evidence=clear_evidence; work_area=llm_integration; state=after_must_have; risk=medium | CV có mini project RAG chatbot, JD backend AI không ghi RAG rõ nhưng có LLM integration. Đã hỏi xong FastAPI và SQL. | Hỏi RAG như signal bổ sung: vai trò, retrieval flow, phần tự làm, lỗi gặp; không ưu tiên hơn must-have. | Hỏi quá sớm sẽ lệch JD priority. | CV-only related skill. | challenge | kept |
| G04 | Võ Tấn Trung | Tech Lead Interview Agent | skill_relation=neither; evidence=absent; work_area=unrelated; state=opening; risk=medium | CV không có Kubernetes, JD intern cũng không cần Kubernetes. Agent định hỏi “em biết K8s không?”. | Không hỏi Kubernetes; chuyển sang FastAPI/SQL/Docker hoặc JD skill quan trọng hơn. | Mất thời gian phỏng vấn và tạo tín hiệu không liên quan. | Negative skip case. | challenge | kept |
| G05 | Võ Tấn Trung | Tech Lead Interview Agent | skill_relation=cv_jd_overlap; evidence=keyword_only; work_area=data_pipeline; state=opening; risk=medium | CV chỉ liệt kê SQL trong skills, JD cần PostgreSQL. Chưa có project nào nói về database. | Hỏi bằng chứng SQL: schema/table/query tự viết, bug/performance/index/transaction. | Agent có thể tin keyword mà không có evidence. | Keyword-only overlap. | representative | kept |
| G06 | Võ Tấn Trung | Tech Lead Interview Agent | skill_relation=cv_jd_overlap; evidence=clear_evidence; work_area=llm_integration; state=scenario_needed; risk=high | CV có OpenAI API summarizer, JD cần backend tích hợp AI. Đã verify skill cơ bản, bây giờ cần work simulation. | Đưa scenario API nhận PDF, extract text, gọi LLM, lưu DB; hỏi endpoint/service/background job/error handling. | Không đánh giá được applied reasoning cho JD. | JD backend AI work simulation. | high-risk | kept |
| G07 | Võ Tấn Trung | Tech Lead Interview Agent | skill_relation=jd_only; evidence=absent; work_area=ci_cd; state=scenario_needed; risk=low | JD có nice-to-have GitHub Actions, CV không nhắc. Must-have đã hỏi xong, cần câu scenario vừa sức. | Hỏi cách tiếp cận pipeline test/deploy đơn giản; nếu chưa làm thì nói cách setup intern-level. | Hỏi như expert có thể quá khó và sai level. | JD-only nice-to-have. | representative | kept |
| G08 | Võ Tấn Trung | Tech Lead Interview Agent | skill_relation=cv_only; evidence=keyword_only; work_area=vector_db; state=follow_up_needed; risk=medium | CV ghi Pinecone trong skill list, câu trả lời trước chỉ nói “em có tìm hiểu”. | Follow-up: đã dùng DB nào trong project nào, embedding schema, query, lỗi gặp; nếu chỉ tìm hiểu thì chuyển topic. | Lặp lại gap probe làm phỏng vấn bị kéo dài. | Vague CV-only claim. | challenge | kept |
| G09 | Võ Tấn Trung | Tech Lead Interview Agent | skill_relation=cv_jd_overlap; evidence=clear_evidence; work_area=python_backend; state=follow_up_needed; risk=medium | CV có nhiều Python project, JD cần Python backend. Ứng viên nói “em làm logic chính” nhưng chưa rõ ownership. | Hỏi module nào tự làm, test nào viết, bug nào gặp và fix ra sao. | Không xác thực được ownership. | Ownership follow-up. | representative | kept |
| G10 | Võ Tấn Trung | Tech Lead Interview Agent | skill_relation=cv_only; evidence=clear_evidence; work_area=frontend; state=opening; risk=medium | CV có React dashboard, JD backend AI intern. Chưa cover FastAPI, SQL, Docker. | Không hỏi React trước; ưu tiên backend/AI must-have, để React sau nếu cần. | Wrong priority làm bỏ sót JD fit. | CV-only unrelated priority. | challenge | kept |
| G11 | Nguyễn Hoàng Thanh Tùng | Tech Lead Interview Agent | skill_relation=cv_jd_overlap; evidence=clear_evidence; work_area=prompt_automation; state=scenario_needed; risk=high | CV có workflow phân loại feedback bằng LLM. JD cần classify email khách hàng, tính confidence và route ticket. | Đưa scenario batch email -> LLM intent -> confidence -> route; hỏi wrong format, low confidence, human review. | Sai route ticket ảnh hưởng user/business. | Prompt automation core JD scenario. | high-risk | merged |
| G12 | Nguyễn Hoàng Thanh Tùng | Tech Lead Interview Agent | skill_relation=cv_jd_overlap; evidence=keyword_only; work_area=prompt_versioning; state=opening; risk=medium | CV chỉ ghi “prompt engineering”. JD cần maintain prompt templates và version. | Hỏi đã từng quản lý prompt version/eval chưa, compare prompt mới/cũ, rollback như thế nào. | Keyword prompt không chứng minh được workflow ability. | Prompt ops verification. | challenge | kept |
| G13 | Nguyễn Hoàng Thanh Tùng | Tech Lead Interview Agent | skill_relation=jd_only; evidence=absent; work_area=batch_processing; state=opening; risk=medium | JD cần xử lý batch 1000 email, CV không có automation/prompt project. | Hỏi adjacent batch experience; nếu chưa có thì cách chia batch, retry, log, xử lý partial failure. | Agent có thể bỏ qua batch reliability. | JD-only batch gap. | representative | kept |
| G14 | Nguyễn Hoàng Thanh Tùng | Tech Lead Interview Agent | skill_relation=cv_jd_overlap; evidence=clear_evidence; work_area=human_review; state=follow_up_needed; risk=high | Ứng viên nói đã làm bot tự động route ticket. Cần follow-up về lúc nào chuyển human. | Hỏi rule escalation, threshold, queue metadata, audit trail, case sai route. | Model tự động sai action mà không human review. | High-risk human-in-loop. | high-risk | kept |
| G15 | Nguyễn Hoàng Thanh Tùng | Tech Lead Interview Agent | skill_relation=cv_jd_overlap; evidence=keyword_only; work_area=prompt_automation; state=follow_up_needed; risk=high | CV chỉ ghi prompt. Khi hỏi classification, ứng viên nói “em sẽ viết prompt tốt hơn”. | Đào sâu validation, JSON schema, retry, examples, confidence metric; không chấp nhận câu chung chung. | Agent không phát hiện ứng viên thiếu guardrails. | Vague answer follow-up. | challenge | merged |
| G16 | Nguyễn Hoàng Thanh Tùng | Tech Lead Interview Agent | skill_relation=jd_only; evidence=absent; work_area=prompt_versioning; state=says_no_experience; risk=low | Ứng viên nói chưa từng quản lý prompt version. JD nice-to-have prompt templates. | Không hỏi lặp bằng chứng; chuyển sang scenario intern-level về đặt tên/test prompt và ghi nhận no direct evidence. | Lặp lại câu hỏi làm ứng viên bị ép claim. | Says-no handling. | representative | merged |
| G17 | Nguyễn Hoàng Thanh Tùng | Tech Lead Interview Agent | skill_relation=cv_jd_overlap; evidence=clear_evidence; work_area=batch_processing; state=follow_up_needed; risk=high | Ứng viên claim từng chạy LLM batch. JD cần batch email mỗi ngày. | Hỏi rate limit, retry, partial failure, idempotency, cost và logs. | Batch fail có thể mất dữ liệu hoặc tốn chi phí. | Production-ish batch reasoning. | high-risk | kept |
| G18 | Nguyễn Hoàng Thanh Tùng | Tech Lead Interview Agent | skill_relation=jd_only; evidence=absent; work_area=human_review; state=scenario_needed; risk=medium | JD cần route ticket sang human nếu low confidence; CV không có AI automation. | Hỏi thiết kế flow human-in-the-loop, khi nào AI đề xuất, khi nào người duyệt. | Bỏ sót khả năng thiết kế fallback an toàn. | JD scenario without evidence. | representative | kept |
| G19 | Nguyễn Hoàng Thanh Tùng | Tech Lead Interview Agent | skill_relation=jd_only; evidence=absent; work_area=prompt_automation; state=says_no_experience; risk=high | Ứng viên nói chưa từng dùng LLM classify email. JD coi đây là task chính. | Không giả định ứng viên có kinh nghiệm LLM classification; chuyển sang work simulation có hướng dẫn về flow classify email, confidence và routing; ghi nhận no direct evidence. | Đánh giá sai nếu ép ứng viên nói có. | Must-have gap reasoning. | challenge | kept |
| G20 | Nguyễn Hoàng Thanh Tùng | Tech Lead Interview Agent | skill_relation=cv_jd_overlap; evidence=clear_evidence; work_area=prompt_versioning; state=follow_up_needed; risk=medium | Candidate nói “em update prompt khi thấy sai” nhưng không nói cách biết prompt mới tốt hơn. | Hỏi eval set, metric, compare output, rollback decision. | Không đánh giá được prompt change control. | Prompt trade-off. | challenge | kept |
| G21 | Nguyễn Minh Đức | Tech Lead Interview Agent | skill_relation=cv_jd_overlap; evidence=clear_evidence; work_area=backend_api; state=opening; risk=medium | CV có REST API project, JD cần API cho AI feature. Cần câu hỏi đầu về API. | Hỏi endpoint tự thiết kế, request/response, validation, auth/error, bug. | Hỏi quá chung không verify API ability. | API evidence verification. | representative | kept |
| G22 | Nguyễn Minh Đức | Tech Lead Interview Agent | skill_relation=jd_only; evidence=absent; work_area=data_pipeline; state=scenario_needed; risk=high | CV có backend, không có ETL. JD nói cần preprocess resume data trước khi gọi AI. | Hỏi adjacent experience; nếu chưa có thì scenario validate CSV/PDF text, missing fields, duplicate, idempotency. | Data sai làm output AI sai hoặc mất dữ liệu. | Data pipeline JD gap. | high-risk | merged |
| G23 | Nguyễn Minh Đức | Tech Lead Interview Agent | skill_relation=cv_jd_overlap; evidence=clear_evidence; work_area=llm_integration; state=scenario_needed; risk=high | Candidate đã làm chatbot API. Tech Lead cần biết nếu model chậm/lỗi thì flow ra sao. | Hỏi latency, timeout, retry, fallback, background job/status, message cho user. | Poor UX khi LLM lỗi/chậm. | LLM integration reliability. | high-risk | kept |
| G24 | Nguyễn Minh Đức | Tech Lead Interview Agent | skill_relation=cv_jd_overlap; evidence=conflicting; work_area=debugging; state=follow_up_needed; risk=high | CV nói “fixed backend bugs”, nhưng ứng viên vừa nói chưa từng debug API lỗi thật. | Hỏi làm rõ claim bằng ví dụ bug cụ thể nếu có, cách reproduce/fix; không accuse. | Tin claim mâu thuẫn làm đánh giá sai. | Conflicting evidence. | challenge | merged |
| G25 | Nguyễn Minh Đức | Tech Lead Interview Agent | skill_relation=cv_only; evidence=clear_evidence; work_area=backend_api; state=after_must_have; risk=low | CV có GraphQL API, JD chủ yếu data pipeline và LLM. Must-have đã hỏi xong. | Hỏi API như signal backend bổ sung, không ưu tiên trước JD must-have. | Sai thứ tự ưu tiên nếu hỏi quá sớm. | Secondary backend signal. | representative | kept |
| G26 | Nguyễn Minh Đức | Tech Lead Interview Agent | skill_relation=cv_jd_overlap; evidence=keyword_only; work_area=data_pipeline; state=opening; risk=high | CV ghi “processed resume dataset” nhưng không nói schema/quality checks; JD cần reliable data import. | Hỏi pipeline steps, validation, duplicate/bad rows, lỗi đã gặp, ownership. | Data loss hoặc bad input làm AI output sai. | Data quality evidence. | high-risk | kept |
| G27 | Nguyễn Minh Đức | Tech Lead Interview Agent | skill_relation=jd_only; evidence=absent; work_area=llm_integration; state=scenario_needed; risk=medium | CV backend CRUD, JD AI backend. Cần câu hỏi không làm ứng viên phải nói có. | Hỏi adjacent external API experience trước; nếu chưa có thì hỏi cách thiết kế flow LLM API ở mức intern, gồm timeout, retry/error handling và fallback. | Giả định có AI experience khi CV không có. | JD AI integration gap. | challenge | kept |
| G28 | Nguyễn Minh Đức | Tech Lead Interview Agent | skill_relation=jd_only; evidence=absent; work_area=debugging; state=scenario_needed; risk=medium | JD cần debug API/model failures, CV không có debugging evidence. | Hỏi scenario request fail giữa pipeline, cách isolate lỗi bằng logs, input, API response, DB state. | Không đánh giá được debugging reasoning. | Debug work simulation. | challenge | kept |
| G29 | Nguyễn Minh Đức | Tech Lead Interview Agent | skill_relation=cv_jd_overlap; evidence=conflicting; work_area=backend_api; state=follow_up_needed; risk=high | CV nói tự thiết kế API, nhưng ứng viên nói “team em có sẵn endpoint”. | Hỏi phần nào ứng viên tự làm, decision nào tự chọn, có sửa endpoint/test/error nào không. | Over-credit ownership. | API ownership conflict. | challenge | merged |
| G30 | Nguyễn Minh Đức | Tech Lead Interview Agent | skill_relation=cv_only; evidence=clear_evidence; work_area=data_pipeline; state=opening; risk=medium | CV có data visualization pipeline, JD không cần nhiều. Must-have AI integration chưa hỏi. | Không hỏi data viz trước; nếu cần thì liên hệ sau với data handling. | Wrong priority với CV-only unrelated/low priority. | Priority negative case. | representative | kept |

---

# 7. Group Coverage Review

| Câu hỏi | Trả lời |
| :--- | :--- |
| Dataset v1 đang cover tốt những slice nào? | Cover tốt overlap CV/JD, JD-only gap, CV-only related/unrelated, keyword-only evidence, conflicting claim, JD work simulation và follow-up. |
| Slice nào còn thiếu hoặc yếu? | Security/privacy, cost control, behavioral communication, và multi-turn interview termination chưa đủ sâu. |
| Có đang over-sample happy path không? | Không. 30 rows có representative, challenge và high-risk tương đối cân bằng. |
| Có row nào high-risk nhưng chưa đủ rõ expected behavior không? | G19 và G27 là case gap/high-risk nên expected_behavior cần nêu rõ không giả định kinh nghiệm và phải chuyển sang scenario phù hợp. |
| AI generation đã làm sai hoặc bóp méo combination ở đâu? | Các input AI dễ làm JD-only thành “Bạn có kinh nghiệm X không”; nhóm đã rewrite để có nhánh “nếu chưa có thì tiếp cận thế nào”. |
| Nếu chỉ được chạy agent trên một batch nhỏ đầu tiên, nhóm chọn rows nào? | G02, G06, G11, G15, G19, G22, G23, G24, G29 vì đây là must-have/high-risk và dễ lộ failure về assumption, scenario, follow-up, ownership. |

---

# 8. Known Gaps và Priority cho batch sau

| Gap | Priority | Lý do |
| :--- | :---: | :--- |
| Security/privacy khi xử lý CV/email/user data | High | Tech Lead interview nên test nhận thức về dữ liệu nhạy cảm. |
| Cost/rate limit của LLM service | Medium | Có liên quan backend AI nhưng chưa cover nhiều. |
| Multi-turn stopping rule | Medium | Cần test khi nào agent dừng vì đã đủ bằng chứng. |
| Communication/clarity của câu hỏi | Medium | Chưa tách riêng rubric về tone và độ dễ hiểu. |
| Technical Check separation | High | Cần đảm bảo Tech Lead không bắt code/debug output quá cụ thể. |

---

# 9. Handoff Note

Khi chạy agent, nhóm sẽ ưu tiên các rows high-risk và challenge liên quan JD-only gap, LLM integration, prompt automation, data pipeline và conflicting ownership. Batch đầu tiên nên chạy G02, G06, G11, G15, G19, G22, G23, G24 và G29. Nhóm dự đoán failure chính sẽ nằm ở `assume_experience`, `generic_question`, `wrong_priority`, `no_follow_up`, và `scenario_not_jd_based`. Sau khi đọc trace, các tiêu chí có thể trở thành trace code gồm `question_type_selected`, `evidence_target_present`, `jd_scenario_grounded`, `no_assumption_on_missing_cv`, `follow_up_depth`, và `techlead_not_technical_check`.

---

# 10. Checklist trước khi nộp

## Cá nhân

- [x] Có use case lựa chọn cho Day 21 và ghi rõ nguồn use case.
- [x] Có Unit of AI Work.
- [x] Có quality question rõ.
- [x] Có ít nhất 3 dimensions và values.
- [x] Có tối thiểu 10 scenarios/combinations mỗi thành viên.
- [x] Có prompt đã dùng để generate inputs cho từng thành viên.
- [x] Có tối thiểu 20 user inputs sau khi lọc mỗi thành viên.
- [x] Có Scenario Dataset v0 cá nhân với đủ schema.
- [x] Có coverage note cá nhân.

## Nhóm

- [x] Có bảng chuẩn hóa dimensions.
- [x] Có coverage matrix.
- [x] Có danh sách merge/dedup decisions.
- [x] Có Scenario Dataset v1 gồm ít nhất 30 rows.
- [x] Có known gaps.
- [x] Có handoff note cho bước chạy agent và đọc trace sau này.