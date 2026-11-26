---
title: "Truy vấn Dữ liệu & Hệ thống Tìm kiếm trong RAG"
description: "Tổng hợp lý thuyết và vận hành Data Retrieval trong RAG: BM25, Learning to Rank, Dense Retrieval, ColBERT, Cross-Encoder, GraphRAG, RRF và đánh giá hệ thống."
date: 2026-11-26 17:54:00 +0700
categories: [AI & Agent, RAG]
tags: [rag, data-retrieval, bm25, semantic-search, graphrag, reranking]
pin: false
toc: true
math: true
---

## 1. Tổng quan

**Truy vấn dữ liệu (Data Retrieval)** là quá trình trích xuất thông tin cụ thể và phù hợp nhất từ một kho dữ liệu lớn dựa trên nhu cầu tìm kiếm (query) của người dùng hoặc hệ thống. Trong kỷ nguyên AI hiện đại — đặc biệt là kiến trúc **RAG (Retrieval-Augmented Generation)** — chất lượng của bước truy vấn quyết định trực tiếp đến độ chính xác của câu trả lời đầu ra, vì mô hình ngôn ngữ lớn (LLM) chỉ có thể tổng hợp tốt khi được cung cấp đúng ngữ cảnh.

Dữ liệu cần truy vấn thường chia làm 3 nhóm chính:
- **Dữ liệu có cấu trúc (Structured Data):** Lưu theo lược đồ chặt chẽ như các bảng trong cơ sở dữ liệu SQL.
- **Dữ liệu phi cấu trúc (Unstructured Data):** Văn bản thuần, tài liệu PDF, hình ảnh, âm thanh.
- **Dữ liệu bán cấu trúc (Semi-structured Data):** Có thẻ hoặc cấu trúc lồng nhau linh hoạt như HTML, JSON, XML.

Tùy vào bài toán thực tế, hệ thống truy vấn có thể phục vụ tìm kiếm đoạn văn bản (Passage Retrieval), chuyển câu hỏi tự nhiên thành truy vấn SQL (Text-to-SQL), tìm kiếm hình ảnh (Image Retrieval), hoặc tìm kiếm đa phương thức (Multimodal Search — kết hợp đồng thời cả văn bản và hình ảnh).

### Xu hướng truy vấn phổ biến hiện nay

Trong thực tế triển khai các hệ thống tìm kiếm và RAG hiện đại, không có một phương pháp đơn lẻ nào giải quyết tốt mọi trường hợp. Các mô hình phổ biến nhất hiện nay thường đi theo hướng kết hợp nhiều tầng:

1. **Hybrid Search + Cross-Encoder Reranking (Chuẩn mực phổ biến nhất trong RAG):** Chạy song song tìm kiếm từ khóa (**BM25**) và tìm kiếm ngữ nghĩa (**Dense Vector Search**), gộp kết quả bằng thuật toán **RRF (Reciprocal Rank Fusion)** rồi lọc lại top kết quả bằng **Cross-Encoder Reranker**. Cách này cân bằng tốt giữa tốc độ, khả năng hiểu ngữ nghĩa và độ chính xác với tên riêng/mã kỹ thuật.
2. **Dense Retrieval kết hợp Metadata Filtering (Phổ biến cho hệ thống vừa và nhỏ):** Dùng Embedding Model kèm bộ lọc siêu dữ liệu (ngày tạo, nguồn, quyền truy cập) trên các Vector Database (như Qdrant, Milvus, pgvector) khi ưu tiên tốc độ triển khai và chi phí thấp.
3. **Late Interaction / Multi-Vector — ColBERT:** Ngày càng được ưa chuộng trong các bài toán đòi hỏi độ chính xác rất cao ở cấp độ từ khóa chi tiết mà vẫn giữ được khả năng hiểu ngữ cảnh.
4. **GraphRAG & Agentic Retrieval:** Được dùng cho các bài toán suy luận đa bước, câu hỏi tổng hợp toàn kho dữ liệu (GraphRAG), hoặc khi cần LLM tự động lập kế hoạch, viết lại câu hỏi và truy vấn lặp nhiều vòng (**Agentic RAG**).
5. **BM25 + Learning to Rank (LTR):** Vẫn là trụ cột tại các hệ thống tìm kiếm công nghiệp truyền thống (thương mại điện tử, web search) nhờ tốc độ cực nhanh và khả năng kết hợp mượt mà các tín hiệu phi văn bản như độ mới, độ phổ biến hay hành vi click.

---

## 2. Các thành phần và phương pháp chính

Một hệ thống truy vấn hiện đại được xây dựng từ nhiều phương pháp tiếp cận khác nhau, từ so khớp mặt chữ truyền thống đến hiểu ngữ nghĩa sâu và suy luận trên đồ thị.

### 2.1. Truy vấn so khớp từ khóa (Keyword / Lexical Search)

Phương pháp truyền thống hoạt động dựa trên việc so khớp chính xác các từ xuất hiện trong câu truy vấn và văn bản. Để đạt tốc độ tìm kiếm tức thì trên hàng triệu tài liệu, hệ thống lưu trữ dữ liệu dưới dạng **chỉ mục ngược (Inverted Index)**: với mỗi từ $t$ trong từ điển, hệ thống lưu giá trị **IDF** của từ đó kèm danh sách các `doc_id` chứa $t$ và tần suất xuất hiện (**TF**) tương ứng.

- **TF (Term Frequency):** Đo mức độ xuất hiện của một từ trong văn bản, thường được chuẩn hóa theo độ dài văn bản để tránh thiên vị các tài liệu quá dài.
- **IDF (Inverse Document Frequency):** Đo độ hiếm của từ trên toàn bộ kho dữ liệu theo công thức $\log\left(\frac{\text{tổng số văn bản}}{\text{số văn bản chứa từ đó}}\right)$ — từ càng hiếm thì trọng số phân biệt càng cao.

Khi xếp hạng, mô hình **TF-IDF** truyền thống biểu diễn câu truy vấn và văn bản thành các vector trọng số $\text{TF} \times \text{IDF}$ trong không gian vector (Vector Space Model), rồi tính **Cosine Similarity** để đo độ tương đồng. Trong thực tế, người ta thường dùng **BM25 (Okapi BM25)** — một mô hình xác suất cải tiến từ TF-IDF. Thay vì tính Cosine Similarity, BM25 cộng trực tiếp điểm xác suất liên quan của từng từ trùng khớp $t \in q \cap d$, đồng thời bổ sung cơ chế **bão hòa TF** (từ lặp quá nhiều thì điểm tăng chậm lại và bị chặn trần để chống spam) và **chuẩn hóa độ dài văn bản**:

$$\text{BM25}(q, d) = \sum_{t \in q \cap d} \text{IDF}(t) \cdot \frac{\text{tf}(t, d) \cdot (k_1 + 1)}{\text{tf}(t, d) + k_1 \cdot \left(1 - b + b \cdot \frac{|d|}{\text{avgdl}}\right)}$$

*(Trong đó: $k_1 \approx 1.2\text{--}2.0$ kiểm soát tốc độ bão hòa TF; $b \approx 0.75$ kiểm soát mức phạt độ dài với $|d|$ là độ dài văn bản và $\text{avgdl}$ là độ dài trung bình của cả kho).*

Dù không hiểu được từ đồng nghĩa hay ngữ cảnh, Keyword Search (tiêu biểu là **Elasticsearch** trên lõi Apache Lucene) vẫn không thể thay thế khi cần tìm chính xác tên riêng, mã lỗi, số hiệu hoặc thuật ngữ đặc thù.

### 2.2. Truy vấn bằng học máy (Learning to Rank — LTR)

**Learning to Rank** chuyển bài toán xếp hạng kết quả tìm kiếm thành bài toán học máy có giám sát. Hệ thống trích xuất tập đặc trưng của từng cặp `(Query, Document)` làm đầu vào (như điểm BM25, TF-IDF, độ dài văn bản, tỉ lệ khớp tiêu đề, hành vi click...) và huấn luyện theo nhãn mức độ liên quan (nhị phân `0/1` hoặc thang điểm `0–3`).

LTR thường chia theo 3 hướng tối ưu hàm mất mát:
- **Pointwise:** Chấm điểm liên quan độc lập cho từng văn bản (ví dụ: Linear Regression, Logistic Regression).
- **Pairwise:** Học cách so sánh xem văn bản nào liên quan hơn trong từng cặp văn bản (ví dụ: RankSVM).
- **Listwise:** Tối ưu trực tiếp thứ tự của cả danh sách kết quả theo các độ đo xếp hạng như nDCG hoặc MAP (ví dụ: các mô hình Ensemble **GBDT / LambdaMART** trên XGBoost, LightGBM).

Do tốn công thiết kế đặc trưng thủ công (feature engineering) và hiểu ngữ nghĩa câu chữ kém hơn Transformer, LTR ít được dùng làm bộ truy xuất chính trong RAG, nhưng vẫn rất mạnh ở tầng **Re-ranking** của các công cụ tìm kiếm lớn nhờ tốc độ cao và dễ trộn tín hiệu hành vi người dùng.

### 2.3. Truy vấn ngữ nghĩa (Semantic Search / Dense Retrieval)

Thay vì khớp mặt chữ, Semantic Search sử dụng các mô hình ngôn ngữ / **Embedding Model** (như BERT, RoBERTa, BGE, E5...) để mã hóa câu truy vấn và văn bản thành các **vector dày đặc (dense embeddings)** trong không gian ngữ nghĩa, sau đó đo khoảng cách bằng **Cosine Similarity** hoặc **Dot Product**. Nhờ vậy, hệ thống hiểu được ý nghĩa tương đương và từ đồng nghĩa ngay cả khi không trùng từ khóa.

Các thành phần cốt lõi trong nhánh Semantic Search gồm:

- **Kiến trúc Bi-Encoder & Domain Adaptation:** Để thích nghi với từ vựng chuyên ngành, mô hình thường được fine-tune theo kiến trúc **Bi-Encoder** gồm 2 nhánh mã hóa độc lập cho `Query` và `Document` — có thể dùng chung trọng số (**Siamese**) hoặc 2 bộ trọng số riêng (**Dual-Encoder / Asymmetric Bi-Encoder**). Quá trình huấn luyện thường áp dụng **Contrastive Learning / Triplet Loss** trên bộ ba `(Query, Positive Doc, Negative Doc)` để kéo câu hỏi lại gần văn bản đúng và đẩy xa văn bản sai. *(Các kỹ thuật pre-train như **Condenser / coCondenser** còn giúp nén thông tin toàn đoạn văn vào vector `[CLS]` hiệu quả hơn).*
- **Tìm kiếm lân cận xấp xỉ (ANN — Approximate Nearest Neighbor):** Với hàng triệu vector, quét vét cạn (brute-force) quá chậm nên cần các cấu trúc chỉ mục xấp xỉ:
  - **IVF (Inverted File Index):** Dùng K-Means (hoặc Hierarchical K-Means) chia không gian vector thành các cụm; khi truy vấn chỉ so sánh với các tâm cụm gần nhất rồi mới quét bên trong cụm đó.
  - **HNSW (Hierarchical Navigable Small World):** Nối các vector thành đồ thị lân cận nhiều tầng (tầng trên thưa để nhảy xa nhanh, tầng dưới dày để tìm chính xác), mang lại tốc độ nhanh và độ phủ (recall) rất cao nhưng tốn nhiều RAM.
- **Tương tác muộn — ColBERT (Late Interaction):** Khắc phục nhược điểm "nén cả đoạn văn dài vào 1 vector duy nhất" của Bi-Encoder bằng cách giữ lại vector cho **từng token** (multi-vector). Khi truy vấn, với mỗi token của `Query`, hệ thống tính **MaxSim** (độ tương đồng lớn nhất với mọi token của `Doc`) rồi cộng tổng lại. Cách này đạt độ chính xác rất cao và giữ được chi tiết từ khóa, nhưng tốn nhiều RAM/ổ cứng để lưu vector theo token.
- **Xếp hạng lại — Cross-Encoder (Full Interaction):** Ghép cặp `[CLS] Query [SEP] Document [SEP]` đưa qua BERT cùng lúc để các token attention chéo qua mọi tầng trước khi chấm điểm liên quan. Reranker có thể huấn luyện theo 3 hướng hàm mất mát mạng neural: **Pointwise** (BCE hoặc MSE trên từng cặp `(Query, Document)`), **Pairwise** (Margin Ranking Loss hoặc Pairwise Cross-Entropy so sánh `Doc+` và `Doc-`), hoặc **Listwise** (InfoNCE, ListNet hoặc ListMLE tối ưu trên cả danh sách ứng viên). Vì phải chạy mô hình lại cho từng cặp `(Query, Document)`, Cross-Encoder quá chậm để tìm trên toàn kho nên chỉ dùng ở bước **Rerank** cho top 20–100 kết quả từ bước truy xuất thô.

### 2.4. GraphRAG (RAG kết hợp Đồ thị tri thức)

**GraphRAG** bổ sung một **Đồ thị tri thức (Knowledge Graph)** lưu các **Thực thể (Entities)** và **Mối quan hệ (Relationships)** song song với các đoạn văn bản rời rạc. Cách tiếp cận này giải quyết 2 điểm yếu lớn của Vector RAG truyền thống: khả năng **suy luận đa bước (Multi-hop reasoning)** qua nhiều thực thể trung gian ($A \to B \to C$) nằm rải rác ở nhiều tài liệu, và các **câu hỏi tổng hợp toàn cục (Global queries)** đòi hỏi nhìn bức tranh toàn cảnh thay vì chỉ đọc top-$k$ đoạn cục bộ.

Quy trình tiêu biểu (như Microsoft GraphRAG) gồm 2 giai đoạn:
1. **Indexing:** Dùng LLM quét qua các chunks để trích xuất thực thể và quan hệ $\to$ dùng thuật toán phân cụm đồ thị (như **Leiden**) gom thành các **cộng đồng (Communities)** đa tầng $\to$ dùng LLM tóm tắt sẵn từng cộng đồng.
2. **Querying:** Áp dụng **Local Search** (tìm thực thể khớp câu hỏi và men theo các cạnh lân cận để lấy ngữ cảnh chi tiết cho câu hỏi đa bước) hoặc **Global Search** (tổng hợp câu trả lời từ các bản tóm tắt cộng đồng đã tạo sẵn cho câu hỏi bao quát).

Đổi lại khả năng suy luận phức tạp và liên kết chéo xuất sắc, GraphRAG có chi phí tạo index rất cao, chậm (tốn nhiều token LLM) và khó cập nhật tức thời khi dữ liệu biến động liên tục.

---

## 3. Cách dùng & Vận hành thực tế

Để xây dựng một pipeline truy vấn hiệu quả trong thực tế, các thành phần trên được ghép nối theo trình tự từ tiền xử lý, truy xuất, tối ưu bộ nhớ đến đánh giá và tinh chỉnh tham số.

### 3.1. Chuẩn bị dữ liệu & Pipeline truy xuất chuẩn

1. **Chia nhỏ văn bản (Chunking) & Gắn Metadata:** Trước khi tạo chỉ mục, văn bản dài cần được cắt thành các đoạn vừa phải (`chunk_size` thường từ 256–1024 tokens, kèm `chunk_overlap` 10–20% để không đứt mạch ngữ cảnh). Mỗi chunk nên gắn kèm metadata (`date`, `source`, `category`, `access_role`) để thực hiện **Metadata Filtering** (ưu tiên lọc trước hoặc lọc tích hợp trong lúc tìm kiếm thay vì lọc sau khi lấy top-$k$, tránh bị thiếu kết quả).
2. **Biến đổi câu truy vấn (Query Transformation):** Với câu hỏi người dùng ngắn, mơ hồ hoặc gộp nhiều ý, có thể dùng LLM để viết lại câu hỏi (**Query Rewriting**), tách thành nhiều truy vấn con (**Multi-Query**), hoặc dùng **HyDE (Hypothetical Document Embeddings)** — sinh ra một đoạn trả lời giả định rồi dùng vector của đoạn đó đi tìm tài liệu thật nhằm thu hẹp khoảng cách ngữ nghĩa.
3. **Truy xuất kết hợp (Hybrid Search) bằng RRF:** Chạy đồng thời BM25 và Vector Search để lấy tập ứng viên (ví dụ top 50–100). Vì điểm BM25 (không giới hạn) và Cosine Similarity ($[-1, 1]$) khác thang đo, phương pháp hợp nhất ổn định nhất là **RRF (Reciprocal Rank Fusion)** dựa trên thứ hạng:
   $$\text{RRF}(d) = \sum_{i} \frac{1}{k + \text{rank}_i(d)} \quad (\text{thường chọn } k = 60)$$
4. **Reranking & Nén bộ nhớ (Quantization):** Đưa top ứng viên qua **Cross-Encoder** để sắp xếp lại top 3–10 đoạn chất lượng nhất trước khi gửi cho LLM. Nếu kho dữ liệu lên tới hàng triệu vector khiến HNSW tốn quá nhiều RAM, có thể áp dụng **Scalar Quantization (`int8` / SQ8)**, **Product Quantization (PQ)** hoặc **Binary Quantization** để giảm 4×–32× dung lượng bộ nhớ, sau đó chấm điểm lại trên vector gốc ở bước cuối.

### 3.2. Đánh giá và tinh chỉnh hệ thống

Cần đo lường định lượng để so sánh mô hình và chọn bộ tham số (`top_k`, kích thước chunk, ngưỡng lọc). Thông thường độ phù hợp được đánh giá theo chuẩn nhị phân ("phù hợp" nếu chứa phần thông tin liên quan, hoặc nhiều mức với nDCG), và tránh dùng **Accuracy** vì lượng văn bản không liên quan quá lớn sẽ làm chỉ số cao ảo.

| Nhóm độ đo | Độ đo | Công thức / Ý nghĩa cốt lõi | Khi nào nên dùng |
| :--- | :--- | :--- | :--- |
| **Không xét thứ tự** | **Precision@k** | $\frac{\text{Số văn bản phù hợp trong top } k}{k}$ | Kiểm tra độ "sạch" của top-$k$ kết quả trả về. |
| **Không xét thứ tự** | **Recall@k** | $\frac{\text{Số văn bản phù hợp trong top } k}{\text{Tổng số văn bản phù hợp trong kho}}$ | Đảm bảo không bỏ sót thông tin quan trọng trước khi đưa vào LLM/Rerank. |
| **Không xét thứ tự** | **F-score ($F_1$)** | $\frac{2 \times \text{Precision} \times \text{Recall}}{\text{Precision} + \text{Recall}}$ | Cân bằng giữa Precision và Recall. |
| **Xét thứ tự** | **AP / MAP** | $\text{AP} = \frac{1}{R} \sum_{k=1}^{K} \text{Precision}@k \times \text{rel}_k$; $\text{MAP} = \frac{1}{|Q|} \sum \text{AP}(q)$ | Cần tìm **nhiều** kết quả đúng và xếp chúng lên đầu danh sách. |
| **Xét thứ tự** | **MRR** | $\frac{1}{|Q|} \sum_{q=1}^{|Q|} \frac{1}{\text{rank}_q}$ (nghịch đảo vị trí kết quả đúng đầu tiên) | Chỉ cần **1 kết quả đúng** xuất hiện càng sớm càng tốt. |
| **Xét thứ tự** | **nDCG@k** | $\text{DCG}@k = \sum_{i=1}^{k} \frac{\text{rel}_i}{\log_2(i + 1)}$, $\text{nDCG}@k = \frac{\text{DCG}@k}{\text{IDCG}@k}$ | Kết quả có **nhiều mức độ liên quan** (`0, 1, 2, 3...`), cần chuẩn hóa về $[0, 1]$. |

Ngoài các độ đo truy xuất thuần túy, khi vận hành thực tế cần theo dõi thêm:
- **Tách bạch đánh giá Retrieval và Generation trong RAG:** Sử dụng các chỉ số như **Context Precision** và **Context Recall** (ví dụ qua framework **RAGAS**) để biết lỗi nằm ở khâu tìm kiếm thiếu ngữ cảnh hay do LLM suy luận sai.
- **Kiểm thử A/B (A/B Testing):** Chia một phần nhỏ lưu lượng người dùng thật (ví dụ 1%) sang phiên bản mới để kiểm chứng hiệu quả qua tỉ lệ click và mức độ hài lòng.
- **Tiêu chí vận hành & hiển thị:** Cân đối tốc độ lập chỉ mục (indexing), độ trễ truy vấn (latency), khả năng giải thích kết quả, chi phí hạ tầng/API, và lựa chọn hiển thị **trích đoạn tĩnh** (mở đầu/tóm tắt cố định) hay **trích đoạn động** (cắt đúng đoạn khớp câu truy vấn).
