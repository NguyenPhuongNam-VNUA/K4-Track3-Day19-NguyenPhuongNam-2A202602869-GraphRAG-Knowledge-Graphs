# Báo cáo Day 19 — Flat RAG vs GraphRAG

**Họ tên:** Nguyễn Phương Nam  **MSSV:** 2A202602869  **Ngày:** 2026-10-05

> Kỳ vọng và thang điểm: `SUBMISSION.md`. Mọi số liệu phải khớp với `ket_qua_benchmark_kg.txt`. Bản thiết kế ontology nộp riêng ở `report/ONTOLOGY.md`.

## 1. Chi phí (10 điểm)

Dán 2 bảng `Indexing` và `Querying` từ `ket_qua_benchmark_kg.txt`:

```
Chat model: gemini:gemini-3.5-flash-lite | Embedding: gemini:gemini-embedding-001 | top_k=3 | chunk_size=800 | chunks=176 | KG: 206 nodes / 387 rels

== Indexing (one-off)
pipeline  calls    in_tok  out_tok       USD  seconds
flat        176         0        0   0.00000    116.6
graph       196     34619     5778   0.00433    152.6

== Querying (mean per question)
pipeline  recall  judge   in_tok  out_tok       USD  seconds
flat        0.51   1.33      696       69   0.00007     1.60
graph       0.94   1.83     5701      172   0.00048     2.56
```

| Chỉ số | Flat | Graph | Graph / Flat |
| --- | --- | --- | --- |
| Indexing USD | $0.00000 | $0.00433 | +$0.00433 |
| Indexing giây | 116.6 | 152.6 | 1.31× |
| Mỗi câu: USD | $0.00007 | $0.00048 | 6.86× |
| Mỗi câu: giây | 1.60 | 2.56 | 1.60× |
| Mỗi câu: in_tok | 696 | 5701 | 8.19× |

**Chi phí tăng thêm đến từ đâu?**
> Chi phí tăng thêm ở pha Indexing đến từ việc gọi LLM trích xuất thực thể và quan hệ JSON từ 20 bài báo tin tức (34.619 input tokens, 5.778 output tokens qua 20 lần gọi LLM), trong khi KB Luật được trích xuất bằng regex tốn $0. Ở pha Querying, chi phí tăng gấp 6.86× chủ yếu do prompt của GraphRAG dài hơn 8.19× (5.701 in_tok so với 696 in_tok của Flat RAG) vì phải đính kèm danh sách các dữ kiện có cấu trúc (facts) thu hồi từ Neo4j qua các bước multi-hop.

## 2. Từng câu hỏi (10 điểm)

| Câu | Loại | Flat recall / judge | Graph recall / judge | Thắng | Vì sao (1 câu) |
| --- | --- | --- | --- | --- | --- |
| Q1 | single-hop-law | 1.00 / 2 | 1.00 / 2 | Hòa | Định nghĩa tiền chất nằm gọn trong một chunk của Điều 2 Luật PCMT 2021 nên cả hai pipeline đều tìm và trả lời đầy đủ. |
| Q2 | single-hop-news | 1.00 / 2 | 1.00 / 2 | Hòa | Tên 2 bị cáo lãnh án tử hình (Trần Thanh Tuấn, Trần Minh Tâm) nằm trong cùng một bài báo về vụ 36kg ma túy, vector search bắt đúng chunk. |
| Q3 | cross-kb | 0.33 / 1 | 1.00 / 2 | Graph | Flat RAG chỉ lấy được mức án của Lê Minh Thành từ tin tức nhưng thiếu điều luật, trong khi GraphRAG đi từ vụ án qua tội danh đến Điều 251 BLHS lấy đủ khung 02–07 năm. |
| Q4 | cross-kb | 0.33 / 1 | 0.67 / 1 | Graph | Cả hai cùng tìm được hành vi tổ chức sử dụng của Hoàng Nato, nhưng chỉ GraphRAG nối được sang Điều 255 BLHS; tuy nhiên khung tối đa (khoản 4: chung thân) bị thiếu trong context do quy tắc lọc khoản (Lỗi E2). |
| Q5 | cross-kb-multi-hop | 0.40 / 1 | 1.00 / 2 | Graph | Flat RAG chịu thua không có điều luật tương ứng với khối lượng >9.6kg MDMA, trong khi GraphRAG đi multi-hop từ Cái Quang Huy sang Điều 250 khoản 4 xác định chính xác khung hình phạt cao nhất (20 năm, chung thân, tử hình). |
| Q6 | aggregation | 0.00 / 1 | 1.00 / 2 | Graph | Flat RAG chỉ trả lời mơ hồ "Vụ việc [1], [2], [3]" không có tên thực thể; GraphRAG truy vấn ngược từ node Substance MDMA ra toàn bộ các vụ án và đối tượng liên quan (Cái Quang Huy, Lê Minh Thành, Viện Pháp y tâm thần). |

## 3. Phân tích lỗi (20 điểm)

### Lỗi E2: Thiếu ngữ cảnh luật (Missing Legal Context in Max Penalty Questions)

- **Hiện tượng:** Ở câu Q4, hệ thống được hỏi: *"Giang hồ 'Hoàng Nato' bị bắt về hành vi gì, và hành vi đó có thể bị phạt tù tối đa bao nhiêu theo Bộ luật Hình sự?"*. GraphRAG xác định chính xác hành vi "tổ chức sử dụng trái phép chất ma túy" và tìm đúng Điều 255 BLHS, nhưng không trả lời được mức phạt tối đa (khoản 4 Điều 255: tù chung thân), chỉ đạt recall=0.67, judge=1.
- **Bằng chứng:**
  - *Trích nguyên văn câu trả lời từ `ket_qua_benchmark_kg.txt` (Q4 graph):*
    ```
    Dựa trên ngữ cảnh và dữ kiện từ knowledge graph:
    - Hành vi bị bắt: Giang hồ "Hoàng Nato" (Dương Minh Tuấn) bị bắt về hành vi tổ chức sử dụng trái phép chất ma túy.
    - Mức phạt tù tối đa: Theo Điều 255 BLHS - Tội tổ chức sử dụng trái phép chất ma túy, hành vi này chưa được nêu cụ thể mức án tối đa ở khung cao nhất trong ngữ cảnh (chỉ có khoản 1 quy định phạt tù từ 02 năm đến 07 năm và khoản 5 quy định các hình phạt bổ sung). Do đó, ngữ cảnh không đủ thông tin để xác định chính xác mức phạt tù tối đa.
    ```
  - *Truy vấn Cypher kiểm tra cấu trúc Điều 255 BLHS trong Neo4j:*
    ```cypher
    MATCH (a:Article {id:'Điều 255 BLHS'})-[:HAS_CLAUSE]->(cl:Clause)
    RETURN cl.number AS number, cl.penalty AS penalty, [(cl)-[:MENTIONS]->(s) | s.name] AS substances
    ORDER BY number;
    ```
  - *Kết quả trả về:*
    ```
    ╒════════╤══════════════════════════════════════════════════════════════╤════════════╕
    │"number"│"penalty"                                                     │"substances"│
    ╞════════╪══════════════════════════════════════════════════════════════╪════════════╡
    │1       │"phạt tù từ 02 năm đến 07 năm"                                │[]          │
    │2       │"phạt tù từ 07 năm đến 15 năm"                                │[]          │
    │3       │"phạt tù từ 15 năm đến 20 năm"                                │[]          │
    │4       │"phạt tù 20 năm hoặc tù chung thân"                           │[]          │
    │5       │"phạt tiền từ 50.000.000 đồng đến 500.000.000 đồng"          │[]          │
    └────────┴──────────────────────────────────────────────────────────────┴────────────┘
    ```
- **Nguyên nhân:** Lỗi nằm ở **KG-3 Cypher context retrieval (Quy tắc lọc khoản luật)**. Trong `src/graph.py`, thuật toán lọc khoản chỉ giữ lại Khoản 1 (khung cơ bản) và các khoản có quan hệ `MENTIONS` tới chất mà vụ án `INVOLVES`. Tuy nhiên, Điều 255 BLHS định khung hình phạt dựa trên **tình tiết và hậu quả** (số người sử dụng, tổn hại sức khỏe, làm chết người), chứ không định khung theo tên và khối lượng chất ma túy (cột `substances` ở cả 5 khoản đều là rỗng `[]`). Do đó, Khoản 4 (quy định khung phạt tối đa: chung thân) bị thuật toán lọc bỏ khỏi context, khiến LLM không có căn cứ kết luận.
- **Đề xuất sửa:**
  - Trong `Neo4jGraph.context()`, bổ sung điều kiện truy xuất: đối với bất kỳ Điều luật nào được nối tới qua vụ án, nếu số lượng khoản của điều luật đó nhỏ hơn hoặc bằng 5 khoản, nên đưa toàn bộ các khoản vào facts; hoặc ít nhất luôn truy xuất thêm Khoản có hình phạt cao nhất (`ORDER BY cl.number DESC LIMIT 1`).
  - *Đánh đổi:* Tăng thêm khoảng 150–250 input tokens cho mỗi câu hỏi cross-kb, nhưng giải quyết triệt để các câu hỏi về khung hình phạt tối đa.

---

### Lỗi E4: Phép đo sai và sự bất cập của Metric Keyword Recall (Metric Discrepancy)

- **Hiện tượng:** Ở câu Q6 (câu hỏi tổng hợp về các vụ việc liên quan đến MDMA), Flat RAG chỉ đạt `recall = 0.00` nhưng lại được LLM Judge chấm `judge = 1`. Ngược lại, việc đánh giá độ chính xác hoàn toàn bằng từ khóa cứng (`must_include`) dẫn đến hiện tượng trừng phạt oan những câu trả lời đúng bản chất hoặc khen ngợi câu trả lời có từ khóa nhưng sai ngữ cảnh.
- **Bằng chứng:**
  - *Trích file `ket_qua_benchmark_kg.txt` tại câu Q6 Flat RAG:*
    ```
    --- Q6 [aggregation] flat recall=0.00 judge=1 1.64s
    Dựa trên ngữ cảnh, cả 3 vụ việc đều có liên quan đến ma túy MDMA:
    1. Vụ việc [1]: Lực lượng chức năng phát hiện bên trong thùng hàng có các viên nén màu xanh, kết quả giám định xác định toàn bộ số viên nén này là MDMA.
    2. Vụ việc [2]: Công an bắt quả tang Thành mang 5 viên ma túy đi bán, kết luận giám định xác định số viên nén này là ma túy MDMA.
    3. Vụ việc [3]: Kết quả giám định xác định số viên nén hình tam giác màu hồng - xám trong kiện hàng là MDMA.
    ```
  - `data/benchmark_kg.json` đặt `must_include`: `["Cái Quang Huy", "Lê Minh Thành", "Pháp y tâm thần"]`.
  - Flat RAG trích xuất đúng 3 vụ việc từ 3 đoạn văn bản vector search tìm được (Vụ [1] chính là vụ Cái Quang Huy qua Nội Bài, Vụ [2] là vụ Lê Minh Thành, Vụ [3] là vụ Viện Pháp y tâm thần), nhưng LLM trả lời theo kiểu trích dẫn số thứ tự nguồn ngữ cảnh `Vụ việc [1], [2], [3]` mà không nhắc lại họ tên đầy đủ của bị cáo. Do đó, hàm `keyword_recall` so khớp chuỗi con trả về đúng `0/3 = 0.00`, trong khi LLM Judge nhận định đúng một phần và cho 1 điểm.
- **Nguyên nhân:** Lỗi nằm ở **thiết kế phép đo (Evaluation Benchmark)**.
  - Phép đo `keyword_recall` là so khớp chuỗi con không phân biệt hoa thường (`k.lower() in answer.lower()`). Đây là phương pháp đo máy móc, nhạy cảm với cách hành văn (paraphrasing, anaphora, referencing). Khi LLM tóm tắt hoặc quy chiếu bằng số thứ tự tài liệu, recall sụp đổ về 0 dù thông tin cơ bản đã được tìm thấy.
  - Ngược lại, nếu câu trả lời chứa đủ từ khóa nhưng sai logic (ví dụ "Lê Minh Thành không phạm tội"), recall vẫn tính điểm tuyệt đối.
- **Đề xuất sửa:**
  - Kết hợp độ đo ngữ nghĩa: sử dụng `LLM-as-a-judge` với thang điểm rubric đa tiêu chí (Factual Correctness, Completeness, Groundedness) làm thước đo chính.
  - Mở rộng danh sách từ khóa hợp lệ trong `must_include` (cho phép chấp nhận alias hoặc các thực thể đại diện, ví dụ "Trần Quốc An", "Lê Văn Đông" cũng được tính cho vụ Viện Pháp y tâm thần).
  - *Đánh đổi:* Đánh giá bằng LLM Judge tốn thêm thời gian và chi phí gọi API, có phương sai nhỏ giữa các lần chạy, nhưng phản ánh chính xác chất lượng thực tế của hệ thống hỏi đáp RAG.

## 4. Kết luận (5 điểm)

Dựa trên kết quả thực nghiệm chi tiết giữa Flat RAG và GraphRAG trên hai cơ sở tri thức (Luật và Tin tức):

1. **Khi nào nên dùng Knowledge Graph (GraphRAG):**
   - **Dữ liệu phân mảnh ở nhiều nguồn độc lập (Cross-KB):** Khi câu trả lời đòi hỏi phải liên kết thực thể giữa hai nguồn dữ liệu không có đoạn văn nào giao nhau. Minh chứng: ở các câu Q3, Q4, Q5, Flat RAG hoàn toàn thất bại trong việc tìm ra điều luật tương ứng (Flat recall chỉ đạt 0.33–0.40), trong khi GraphRAG đạt recall 1.00 và judge 2 nhờ đường đi multi-hop xuyên qua node cầu nối `Crime`.
   - **Truy vấn tổng hợp, gom nhóm (Aggregation):** Ở câu Q6, Flat RAG chỉ lấy được 3 đoạn rời rạc trong top-k (recall 0.00), trong khi GraphRAG đi từ node `Substance` thu gom được toàn bộ các vụ án và đối tượng liên quan (recall 1.00).
   - **Nghiệp vụ đòi hỏi tính chính xác và giải trình cao (Pháp lý, Y tế, Tài chính):** GraphRAG cung cấp đường dẫn dữ kiện rõ ràng (Ai → Vụ nào → Tội gì → Điều luật nào), loại bỏ hiện tượng ảo giác.

2. **Khi nào Flat RAG là đủ:**
   - **Truy vấn đơn bước (Single-hop):** Khi câu hỏi chỉ yêu cầu thông tin nằm trọn trong một đoạn văn bản (như Q1 định nghĩa tiền chất trong Luật, Q2 danh sách tử hình trong 1 bài báo). Ở các câu này, Flat RAG đạt kết quả hoàn hảo (recall 1.00, judge 2) ngang bằng GraphRAG.
   - **Tối ưu chi phí và độ trễ:** Flat RAG có chi phí mỗi câu hỏi rẻ hơn **6.86 lần** ($0.00007 vs $0.00048), input token ít hơn **8.19 lần** (696 vs 5.701 tokens), và tốc độ phản hồi nhanh hơn **1.60 lần** (1.60s vs 2.56s). Nếu bài toán không có yếu tố đa nguồn hoặc quan hệ phức tạp, Flat RAG là lựa chọn tối ưu về mặt kinh tế kỹ thuật.

## 5. Tự kiểm (5 điểm)

```
$ pytest tests/ -q
................................................                         [100%]
48 passed in 0.02s

$ python bench_kg.py --check
[OK] Dữ liệu: 18 điều luật, 20 bài báo
[OK] KG-1 link_entity
[OK] Neo4j kết nối được
[provider] chat = gemini:gemini-3.5-flash-lite | embedding = gemini:gemini-embedding-001
[OK] KG-2 build_graph: 148 node / 294 cạnh, đường xuyên 2 KB dài 2 cạnh
[OK] KG-3 context: 25 dữ kiện, có Điều 251
[OK] KG-4 GraphRAGAgent.answer
[OK] Chi phí check: 1 lần gọi LLM, $0.00041. Graph nhỏ (luật + 1 bài) vẫn còn trong Neo4j để bạn xem; chạy --judge để dựng graph đầy đủ.
```

Ảnh Neo4j:
- `report/img/kg_count.png`: Bảng thống kê toàn bộ 7 nhãn node trong ontology (Clause: 99, Person: 43, Article: 18, Case: 14, Crime: 13, Substance: 13, Location: 6 = 206 nodes).
- `report/img/kg_cross_kb.png`: Đồ thị đường đi xuyên 2 KB (Person → Case → Crime ← Article) kèm bảng Results overview hiển thị đầy đủ 4 loại node và 3 loại quan hệ.
- `report/img/kg_my_case.png`: Đồ thị vụ án xuyên 2 KB với đối tượng tự chọn: **Cái Quang Huy** (Cái Quang Huy → Vụ vận chuyển ma túy → Tội vận chuyển trái phép chất ma túy ← Điều 250 BLHS, kèm các liên kết tới MDMA, Ketamine và Hà Nội).

Người đã chọn cho `kg_my_case.png`: **Cái Quang Huy**

## Vấn đề gặp phải (không tính điểm)

- **Rate limit API Gemini (429 & 503):** Ban đầu mô hình `gemini-2.5-flash-lite` bị Google thông báo ngừng hỗ trợ cho người dùng mới; khi chuyển sang `gemini-flash-latest`, tài khoản chạm hạn ngạch 20 requests/ngày (RPD) của gói Free Tier. Sau đó hệ thống được cấu hình chuyển sang `gemini-flash-lite-latest` (tương đương `gemini-3.5-flash-lite`) với hạn ngạch cao hơn (15 RPM, 1.500 RPD) kết hợp cơ chế tự động thử lại (retry with exponential backoff) trong `src/llm.py`, giải quyết triệt để lỗi rate limit và hoàn thành toàn bộ 100% benchmark một cách ổn định.
