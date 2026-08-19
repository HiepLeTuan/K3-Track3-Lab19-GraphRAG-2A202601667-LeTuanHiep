# Báo Cáo Thực Hành & Thuyết Minh Kỹ Thuật — Lab 19: GraphRAG vs Flat RAG

**Học viên:** Lê Tuấn Hiệp  
**Khóa học:** AICB-K34 · Track 3: GraphRAG  
**Ngày thực hiện:** 19/08/2026  

---

## PHẦN 1: THUYẾT MINH KỸ THUẬT & PHÂN TÍCH CA LỖI

### 1. Coreference Resolution (Phân giải đồng tham chiếu)

Một trường hợp khó xuất hiện tại chunk `4f1346392056a403277d::c0000` (source row 33), nơi Aeris Communications và Ericsson cùng xuất hiện trong mô tả giao dịch. Những cụm chung chỉ liên minh/giao dịch hoặc footprint hình thành sau giao dịch có antecedent là cả sự kiện, không phải riêng Aeris hay Ericsson. Nếu coreference resolver ép cụm này về một công ty duy nhất, bước relation extraction có thể thu hẹp sai ngữ nghĩa “joining together” thành quan hệ chủ thể–đối tượng không chính xác. Chính sách conservative hợp lý là giữ nguyên khi antecedent không phải một thực thể đơn nhất.

Hậu quả của false coreference rất nghiêm trọng: bước NER/RE có thể tạo cạnh một chiều sai, gán phát biểu hoặc sự kiện cho nhầm công ty. Sai sót sau đó được Neo4j traversal lan truyền sang các câu hỏi multi-hop. Pipeline giảm rủi ro bằng cách chỉ phân giải khi antecedent xuất hiện rõ trong cùng chunk; trường hợp không chắc chắn được giữ nguyên và ghi vào `unresolved_mentions`.

### 2. Entity Resolution Threshold & Lexical Guard

- Ngưỡng vector matching: cosine similarity `0.90`.
- Lexical Guard chuẩn hóa hậu tố doanh nghiệp và yêu cầu `SequenceMatcher ratio >= 0.72` trước khi Union-Find gộp hai mention.
- Audit thực nghiệm có hai quyết định `MERGE_VECTOR`:
  - `L&T Technology Services` và `L&T Technology Services Limited`: similarity `0.925773`.
  - `Fidelity National Information Services` và `Fidelity National Information Services Inc.`: similarity `0.924524`.

Trong lần chạy này không có cặp nào mang quyết định `REJECT_GUARD`, vì vậy không thể trích dẫn trung thực một cặp similarity trên 0.85 bị chặn từ audit thực tế. Đây là hạn chế của sample: bảng audit chỉ có 2 dòng, thấp hơn yêu cầu 10+ dòng. Một negative control nên bổ sung là `Apple` và `Apple Music`: embedding có thể gần nhau nhưng hai mention biểu diễn công ty và dịch vụ/sản phẩm khác nhau nên không được merge. Hai merge quan sát được đều là biến thể có/không có hậu tố pháp nhân và hợp lý về mặt ngữ nghĩa.

### 3. Đồ thị & Super-node Mitigation

Cell thống kê cuối ghi nhận riêng phiên hiện tại có `197` raw triple, `195` canonical triple và bảng `nodes_df` gồm `330` node. Trong khi đó, sanity check trên Neo4j ghi nhận `634` node, `429` edge và `0` edge thiếu `source_chunk_id` hoặc `published_date`. Chênh lệch cho thấy database có khả năng bao gồm dữ liệu từ lần ingest trước vì không được làm sạch trước khi chạy lại; do đó 634/429 không nên được diễn giải là kích thước thuần của riêng corpus mới.

| Hạng | Tên thực thể | Loại | Degree |
|---:|---|---|---:|
| 1 | Microsoft | Company | 10 |
| 2 | Apple | Company | 9 |
| 3 | NVIDIA | Company | 9 |

Không node nào vượt ngưỡng super-node `degree > 100`, nên policy cap 50 chưa được kích hoạt (`graph_supernode_events = 0` cho cả năm câu). Test trên Microsoft lấy 10/10 edge và không vi phạm giới hạn.

Ưu điểm của cap 50 edge mới nhất là kiểm soát context, latency và token, đồng thời ưu tiên trạng thái hiện hành. Rủi ro là câu hỏi lịch sử có thể mất edge cũ nhưng quan trọng; recency cũng không đồng nghĩa với relevance. Cải tiến là xếp hạng kết hợp thời gian, semantic relevance, relation type và confidence, đồng thời dành quota cho từng giai đoạn.

### 4. So sánh thực nghiệm Flat RAG và GraphRAG

Benchmark sử dụng 5 câu: 1 factoid, 2 multi-hop và 2 cross-doc.

| Tiêu chí | Flat RAG | GraphRAG | Delta (Graph − Flat) | Nhận xét |
|---|---:|---:|---:|---|
| Comprehensiveness (1–5) | 4.600 | 4.200 | −0.400 | Flat cao hơn do GraphRAG sai ở G5000-05. |
| Faithfulness (1–5) | 5.000 | 4.600 | −0.400 | Flat giữ đúng evidence ở cả 5 câu. |
| Multi-hop Reasoning (1–5) | 4.600 | 4.200 | −0.400 | GraphRAG bị relation noise ở một câu multi-hop. |
| Latency trung bình (s) | 1.672 | 1.649 | −0.023 | Gần tương đương; GraphRAG nhanh hơn khoảng 1,4% trong sample. |
| Token usage trung bình | 637.2 | 561.6 | −75.6 | GraphRAG dùng ít hơn khoảng 11,9%. |

Theo nhóm, factoid đạt 5/5 ở cả hai phương pháp. GraphRAG tốt hơn ở cross-doc về comprehensiveness và multi-hop (4,5 so với 4,0), trong khi Flat RAG tốt hơn rõ ở multi-hop (5,0 so với 3,5). Token thấp hơn của GraphRAG là lợi ích thực nghiệm, nhưng chưa đủ bù cho lỗi relation ở `G5000-05`.

#### Ca Flat RAG yếu hơn, GraphRAG thành công

- **Question:** `G5000-02` — phân biệt giao dịch Aeris–Ericsson đang được lên kế hoạch trong hai báo cáo đầu với trạng thái đã hoàn thành trong báo cáo sau.
- **Kết quả:** Flat RAG đạt 4/5 về comprehensiveness và multi-hop; GraphRAG đạt 5/5 trên cả ba tiêu chí.
- **Nguyên nhân Flat RAG yếu hơn:** câu hỏi yêu cầu nối trạng thái sự kiện qua nhiều báo cáo và thời điểm. Các vector chunk độc lập cung cấp evidence nhưng không biểu diễn trực tiếp sự chuyển trạng thái.
- **Cách GraphRAG hỗ trợ:** hybrid context kết hợp các quan hệ có provenance với vector evidence, giúp mô hình nối planned transfer với completed acquisition theo trình tự thời gian.

#### Failure case 2 — GraphRAG không dựng được đường multi-hop

- **Question:** `G5000-05` — đường `Ericsson → Aeris → IoT reach`.
- **Kết quả:** Flat RAG đạt 5/5 ở cả ba tiêu chí; GraphRAG chỉ đạt comprehensiveness 2/5, faithfulness 3/5 và multi-hop 2/5.
- **Nguyên nhân:** evidence nguồn đã có đầy đủ, nhưng schema chỉ cho phép 8 relation và không có `TRANSFERRED_TO`, `SUPPORTS` hoặc `PLANNED_ACQUISITION`. Extraction vì vậy tạo edge `Aeris Communications -PARTNERED_WITH-> Ericsson` từ cụm “joining together”. Graph context đặt edge nhiễu này trước evidence mua lại, khiến generator mô tả sai bước đầu của đường đi.
- **Khắc phục:** căn chỉnh schema hoặc ánh xạ relation đồng nghĩa về relation chuẩn, ví dụ `TRANSFERRED_TO → ACQUIRED` kèm `event_state=planned`; biểu diễn reach bằng node/thuộc tính có provenance; kiểm tra coverage giữa `required_relations` và `ALLOWED_RELATIONS` trước extraction.

Kết quả cho thấy graph traversal chỉ có lợi khi relation extraction và schema biểu diễn đúng ngữ nghĩa. Một edge sai có thể gây hại hơn việc chỉ dùng các vector chunk gốc.

### 5. Trade-offs, kiểm soát AI Coding Agent và scale 350 MB

Flat RAG có pipeline đơn giản, index nhanh và phù hợp factoid/similarity lookup. GraphRAG thêm chi phí coreference, NER/RE, entity resolution, Neo4j ingestion và traversal, nhưng có lợi thế ở câu hỏi nối nhiều document. Benchmark mới cho thấy đúng trade-off này: GraphRAG thắng ở cross-doc nhưng thua ở multi-hop khi extraction gán sai relation. Latency trung bình gần ngang nhau và GraphRAG dùng ít hơn 75,6 token mỗi câu, dù chi phí offline để tạo graph vẫn cao hơn đáng kể.

Trong quá trình debug, tôi không bỏ `validate_golden()` hoặc dùng placeholder để vượt qua cell 4.3 vì như vậy LLM Judge mất correctness anchor. Tôi cũng không tiếp tục dùng model Groq trả 404; pipeline được chuyển sang OpenAI và test request nhỏ trước khi chạy batch.

Khi scale khoảng 350 MB, bottleneck đầu tiên là số lượt LLM cho coreference và extraction, tiếp theo là embedding/entity-resolution và graph size. Hướng xử lý:

1. Dedup và quality filter trước LLM; checkpoint theo `chunk_id` để chạy idempotent.
2. Batch/async worker có rate limiter, retry phân loại và dead-letter queue; không retry lỗi 4xx không thể phục hồi.
3. Entity resolution theo blocking rồi ANN/HNSW, tránh so sánh toàn cục O(N²).
4. Bulk ingestion bằng `UNWIND`, tạo constraint/index trước ingest và tách staging graph khỏi production.
5. Partition/community detection, edge budget theo relation và relevance-aware super-node policy.
6. Version hóa corpus, model, prompt và Golden Dataset để tránh corpus mismatch. Lần chạy mới đã dùng đúng 5.000 dòng đầu và xác nhận đủ evidence row 33, 935, 1746.

---

## PHẦN 2: SUY NGẪM & KẾ HOẠCH ĐỒ ÁN

### 1. Mapping bài giảng vào code

| Khái niệm | Module | Hàm/khối code | Quan sát thực tế |
|---|---|---|---|
| Conservative Coreference | Module 1 | `resolve_coref_batch()`, `run_coref()` | Có fallback về text gốc; cần log lỗi rõ thay vì che giấu exception. |
| Schema & Allowlist Guard | Module 2 | `ALLOWED_NODE_TYPES`, `ALLOWED_RELATIONS`, `run_extraction()` | Tăng precision nhưng có thể làm mất relation cần cho Golden Dataset. |
| Bulk Cypher Ingestion | Module 2 | `bulk_insert_nodes()`, `bulk_insert_edges()` | Dùng `UNWIND`; sanity check xác nhận 0 edge thiếu provenance. |
| Entity Resolution & Union-Find | Module 3 | `build_resolution_map()`, `merge_guard()`, `UF` | Hai merge hậu tố pháp nhân hợp lý, nhưng audit chỉ có 2 dòng và chưa có reject. |
| Flat Vector Retrieval | Module 4 | `build_flat_index()`, `retrieve_flat_context()` | FAISS có 2.105 vector và lấy đúng evidence Aeris–Ericsson. |
| Super-node Degree Cap | Module 4 | `retrieve_graph_context()` | Ngưỡng 100, cap 50, global cap 250; chưa kích hoạt vì degree cao nhất là 10. |
| Hybrid Generation | Module 4 | `answer_graph_rag()` | Có lợi ở cross-doc, nhưng edge `PARTNERED_WITH` gây nhiễu cho G5000-05. |
| LLM-as-a-Judge | Module 5 | `judge_answer()`, `run_evaluation()` | Chạy đủ 5 câu; Flat trung bình 4,6 và GraphRAG 4,2 về quality. |

### 2. Quá trình debugging và bài học

Lỗi khó nhất là chuỗi lỗi liên hoàn ở extraction. Dataset dùng cột `description`, trong khi loader ban đầu chỉ tìm `text/content/article/body/story`. Sau khi sửa loader, model `llama-3.3-70b-versatile` trả 404 cho toàn bộ 200 batch. `run_extraction()` lại trả `pd.DataFrame([])` không có schema, dẫn tới lỗi tại `canonicalize_triples()` khi truy cập `source_raw`.

Tôi xử lý theo từng lớp: bổ sung `description` và `_id`; kiểm tra `extraction_errors_df`; thay provider bằng OpenAI; test request và batch nhỏ trước khi chạy đủ; buộc DataFrame rỗng vẫn có schema và fail sớm nếu không có triple. Tôi cũng thay random sample bằng đúng 5.000 dòng đầu, giữ `source_row_id` và xác nhận các evidence row 33, 935, 1746. Kết quả cuối là 2.105 article/chunk, 197 raw triple, 195 canonical triple, 330 node trong `nodes_df` và 0 batch extraction lỗi.

Bài học chính là không nuốt exception trong batch pipeline, phải validate output tại boundary và luôn chạy smoke test nhỏ trước tác vụ dài. Notebook chạy hết cell chưa đủ; corpus, Golden Dataset và schema relation cũng phải được căn chỉnh.

### 3. Kế hoạch áp dụng vào đồ án thực tế

**Tên đề xuất:** Hệ thống theo dõi quan hệ và diễn biến doanh nghiệp công nghệ.

Bài toán phù hợp Hybrid GraphRAG. Flat retrieval tìm đoạn chi tiết và fact mới chưa extraction; graph traversal phục vụ chuỗi đầu tư, mua bán, đối tác, nhân sự và thay đổi theo thời gian.

- **Nodes:** `Company`, `Person`, `Technology`, `Product`, `Event`, `Article`.
- **Relations:** `ACQUIRED`, `INVESTED_IN`, `FOUNDED`, `WORKED_AT`, `PARTNERED_WITH`, `DEVELOPED`, `USES`, `LEADS`, `PARTICIPATED_IN`, `MENTIONED_IN`.
- **Temporal modeling:** giao dịch quan trọng dùng node `Event` có `announced_date`, `completed_date`, `status` và provenance thay vì ép mọi trạng thái vào một edge.
- **Entity resolution:** alias cho ticker/thương hiệu; blocking theo type; ANN lấy candidate; lexical/type/version guard; human review gần ngưỡng và audit mọi quyết định.
- **Super-node:** cap theo relation, thời gian và relevance; partition theo community/time window, giữ global edge cap mỗi truy vấn.

---

## TỰ ĐÁNH GIÁ

| Tiêu chí | Điểm (1–5) | Ghi chú |
|---|---:|---|
| Mức độ hiểu bài giảng GraphRAG | 4 | Hiểu pipeline và nhận diện corpus/schema mismatch. |
| Khả năng kiểm soát AI Coding Agent | 4 | Kiểm tra lỗi thực tế, không bỏ validation để ép notebook chạy. |
| Chất lượng đồ thị tri thức | 3 | Provenance đạt nhưng audit ít, chưa có super-node và còn relation noise. |
| Khả năng phân tích và debug | 4 | Xử lý loader, model 404, DataFrame schema và runtime reset. |

## Kết luận

Pipeline đã chạy end-to-end trên đúng phạm vi Golden Dataset: 5.000 raw row được lọc còn 2.675 record đủ dài và dedup còn 2.105 article/chunk; extraction 400 chunk tạo 197 raw triple, canonicalization giữ lại 195 triple và tạo 330 node, với 0 batch lỗi; FAISS index 2.105 vector; Neo4j kiểm tra 0 edge thiếu provenance; benchmark xuất đủ hai CSV cho 5 câu. Chất lượng tăng mạnh: Flat RAG đạt trung bình 4,6/5 và GraphRAG 4,2/5 trên ba tiêu chí. GraphRAG tốt hơn ở cross-doc nhưng thua ở multi-hop do relation `PARTNERED_WITH` mô tả sai giao dịch Aeris–Ericsson. Bước tiếp theo là làm sạch Neo4j trước ingest, mở rộng/chuẩn hóa relation schema, tăng audit lên ít nhất 10 dòng và bổ sung negative controls cho Lexical Guard.
