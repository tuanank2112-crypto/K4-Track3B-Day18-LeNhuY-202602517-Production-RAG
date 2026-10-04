# Individual Reflection — Lab 18: Production RAG

**Họ và tên:** Lê Như Ý  
**MSSV:** 202602517  
**Khóa:** K4 - Track 3B  
**Ngày hoàn thành:** 04/10/2026  

---

## Phần 1: Mapping bài giảng (Lecture Mapping)

Bảng đối chiếu toàn diện các khái niệm kiến trúc Production RAG trong bài giảng với mã nguồn thực tế triển khai trong Lab 18:

| Lecture Concept | Module | Hàm cụ thể | Observation & Phân tích chuyên sâu |
|----------------|--------|-------------|------------------------------------|
| **Semantic Chunking** | M1 Chunking | `chunk_semantic()` | Sử dụng mô hình sentence embedding (`all-MiniLM-L6-v2`) tính toán Cosine Similarity giữa các câu liền kề. Với ngưỡng `threshold = 0.85`, văn bản không bị ngắt quãng cơ học giữa câu hay giữa đoạn văn cùng chủ đề (khác với `chunk_basic` cắt cứng theo độ dài hoặc đoạn văn đơn thuần). Kết quả phân đoạn bảo toàn trọn vẹn ngữ cảnh ngữ nghĩa của từng điều khoản chính sách. |
| **Hierarchical Chunking (Parent-Child)** | M1 Chunking | `chunk_hierarchical()` | Tạo cấu trúc phân tầng: Parent chunks (2048 ký tự) chứa bức tranh toàn cảnh của văn bản, trong khi Child chunks (256 ký tự) được sinh ra để đánh chỉ mục vector và tìm kiếm chính xác (high retrieval precision). Khi truy vấn, hệ thống khớp Child chunk trước rồi mở rộng trả về ngữ cảnh của Parent chunk (`parent_id`), giải quyết triệt để bài toán đánh đổi giữa Precision và Context Window. |
| **Structure-Aware Chunking** | M1 Chunking | `chunk_structure_aware()` | Phân tích cấu trúc tiêu đề Markdown (`#`, `##`, `###`), gắn metadata `section` cho từng chunk. Giúp giữ nguyên vẹn cấu trúc bảng biểu, danh sách liệt kê và các điều khoản phụ lục, ngăn chặn việc phân đoạn cắt đôi bảng lương hay bảng quy chuẩn an toàn. |
| **Vietnamese Word Segmentation & BM25** | M2 Search | `segment_vietnamese()`, `BM25Search.index()` & `.search()` | Tiếng Việt có đặc tính từ ghép đa âm tiết. Sử dụng thư viện `underthesea` tách từ chuẩn và thay thế dấu gạch dưới (`_` thành khoảng trắng) để bộ phân tích BM25Okapi không bị sai lệch tokenization giữa tài liệu và câu truy vấn (ví dụ: "nghỉ phép" vs "nghỉ_phép"), đảm bảo độ phủ từ khóa chính xác tuyệt đối đối với các thuật ngữ chuyên môn hoặc mã số chính sách. |
| **Dense Search & Vector Indexing** | M2 Search | `DenseSearch.index()` & `.search()` | Sử dụng embedding đa ngôn ngữ `BAAI/bge-m3` (chiều 1024) kết hợp Vector Database Qdrant (`PointStruct`, khoảng cách Cosine). Khi Docker Qdrant không khả dụng, hệ thống tự động fallback linh hoạt sang Qdrant in-memory client (`:memory:`), cho phép biểu diễn ngữ nghĩa trừu tượng và tìm kiếm tương đồng mượt mà. |
| **Reciprocal Rank Fusion (RRF)** | M2 Search | `reciprocal_rank_fusion()` | Áp dụng công thức chuẩn $RRF(d) = \sum \frac{1}{k + rank + 1}$ với hệ số làm mịn $k = 60$. RRF dung hòa và chuẩn hóa điểm số không đồng nhất giữa BM25 (thang điểm không biên độ) và Dense Cosine (thang điểm 0–1). Các tài liệu xuất hiện đồng thời ở thứ hạng cao ở cả hai danh sách lexical và semantic được đẩy lên vị trí dẫn đầu một cách tự nhiên. |
| **Cross-Encoder Reranking** | M3 Rerank | `CrossEncoderReranker._load_model()` & `.rerank()` | Mô hình `BAAI/bge-reranker-v2-m3` đóng vai trò chốt chặn cuối cùng (Stage 2 reranker), nhận cặp `(query, document_text)` để tính toán cross-attention toàn diện giữa từng từ của truy vấn và tài liệu. Quá trình lọc từ Top 20 candidate xuống Top 3 giúp loại bỏ các văn bản nhiễu (như nhầm lẫn giữa chính sách mật khẩu 90 ngày với quy định nghỉ phép), nâng cao Context Precision rõ rệt với độ trễ tối ưu ~100–250ms. |
| **RAGAS 4 Core Metrics** | M4 Eval | `evaluate_ragas()` | Đánh giá toàn diện 4 trụ cột chất lượng của RAG: **Faithfulness** (đo lường ảo giác/hallucination của LLM từ context), **Answer Relevancy** (mức độ bám sát câu hỏi của câu trả lời), **Context Precision** (tỷ lệ chunks liên quan ở top đầu), và **Context Recall** (mức độ đầy đủ của ngữ cảnh so với Ground Truth). |
| **Diagnostic Tree & Failure Analysis** | M4 Eval | `failure_analysis()` | Phân loại có hệ thống các câu hỏi có điểm số thấp nhất (Bottom-N) theo Cây chẩn đoán (Diagnostic Tree) để xác định chính xác điểm nghẽn nằm ở bước Chunking, Retrieval, Reranking hay Generation, kèm theo giải pháp khắc phục cụ thể. |
| **Enrichment Pipeline (Contextual Prepend & Combined Single-Call)** | M5 Enrichment | `_enrich_single_call()`, `contextual_prepend()` | Bổ sung ngữ cảnh (tóm tắt vị trí chunk trong tài liệu), trích xuất metadata và sinh các câu hỏi giả định (HyQA) trước khi đánh chỉ mục. Tối ưu hóa chi phí và độ trễ bằng kỹ thuật Combined Single-Call (1 LLM call trích xuất JSON toàn bộ thông tin) kèm bộ extractive fallback hoạt động độc lập khi không có kết nối API. |

---

## Phần 2: Khó khăn & Cách giải quyết (Challenges & Debugging)

### 1. Lỗi kỹ thuật gặp phải (Exact Error Messages)

- **Lỗi 1: Tokenization Mismatch trong BM25 Tiếng Việt**
  - *Thông báo / Hành vi lỗi:* Khi tìm kiếm câu truy vấn `"nghỉ phép năm"`, hàm `BM25Search.search()` trả về điểm số `0.0` hoặc trả về kết quả không khớp dù tài liệu gốc có chứa cụm từ này.
  - *Nguyên nhân:* Thư viện `underthesea.word_tokenize(text, format="text")` tự động nối các từ ghép tiếng Việt bằng dấu gạch dưới (ví dụ `"nghỉ_phép"`). Khi BM25 tokenize bằng `split(" ")`, chunk tài liệu chỉ chứa token đơn `"nghỉ_phép"`, trong khi câu truy vấn của người dùng nếu tách bằng `split()` thông thường sẽ thành hai token `"nghỉ"` và `"phép"`. Do đó, BM25 không tìm thấy token khớp chính xác.
  - *Cách sửa:* Trong hàm `segment_vietnamese()`, sau khi tokenize qua underthesea, bắt buộc thực hiện chuỗi chuẩn hóa `.replace("_", " ")` và `.lower()`, đảm bảo cả tài liệu lập chỉ mục và câu truy vấn đều thống nhất một không gian token.

- **Lỗi 2: Qdrant Client Connection Timeout khi không có Docker Daemon**
  - *Thông báo lỗi:* `failed to connect to the docker API at npipe:////./pipe/dockerDesktopLinuxEngine... ResponseHandlingException: Connection refused.`
  - *Nguyên nhân:* Môi trường máy trạm cá nhân chưa khởi chạy Docker Desktop daemon, khiến lời gọi `QdrantClient(host="localhost", port=6333)` bị treo hoặc ném ngoại lệ kết nối.
  - *Cách xử lý:* Tận dụng cơ chế graceful fallback trong `DenseSearch.__init__()`: đặt khối `try ... except` bao quanh việc kết nối tới `localhost:6333` với `timeout=2s`. Khi xảy ra lỗi kết nối, hệ thống tự động khởi tạo `QdrantClient(":memory:")`, cho phép toàn bộ quy trình index và search chạy in-memory mà không làm gián đoạn pipeline. Đồng thời hỗ trợ cả hai phương thức truy vấn `query_points` và `search` của các phiên bản `qdrant-client` khác nhau.

- **Lỗi 3: Transformers / HuggingFace FlagEmbedding Import Crash**
  - *Thông báo lỗi:* Cảnh báo xung đột khi nạp `FlagReranker` với các bản transformers mới do thay đổi cấu trúc `XLMRobertaTokenizer`.
  - *Cách sửa:* Tuân thủ nghiêm ngặt lưu ý kiến trúc trong đề bài: sử dụng `sentence_transformers.CrossEncoder("BAAI/bge-reranker-v2-m3")` thay vì gọi trực tiếp từ `FlagEmbedding`. Mô hình tải nhanh, tương thích tốt và chạy ổn định trên CPU/GPU.

### 2. Kiến thức còn thiếu & Cách khắc phục
- **Đánh giá RAGAS trên tập ngữ liệu Tiếng Việt:** Trước đây em thường đánh giá RAG theo cảm tính hoặc chỉ đo cosine similarity đơn thuần. Qua lab này, em hiểu rõ bản chất 4 chỉ số của RAGAS và nhận ra rằng RAGAS yêu cầu LLM Judge (như GPT-4o-mini) phải phân tích ngữ cảnh tiếng Việt thật chuẩn.
- **RRF vs Weighted Score:** Hiểu được tại sao trong thực tế sản xuất người ta chuộng Reciprocal Rank Fusion hơn việc cộng điểm có trọng số (weighted sum). Lý do là điểm của BM25 phụ thuộc độ dài tài liệu và tần suất từ (không bị chặn trên), trong khi điểm Cosine nằm trong khoảng [-1, 1]; nếu chuẩn hóa không chuẩn (min-max scaling trên dynamic corpus) sẽ dẫn tới méo mó điểm xếp hạng. RRF giải quyết triệt để nhờ xếp hạng tương đối.

---

## Phần 3: Action Plan cho Project cá nhân (Application Plan)

### Dự án: Trợ lý Tra cứu Quy trình & Chính sách Nhân sự Nội bộ Doanh nghiệp (Enterprise HR & Policy Copilot)

#### 1. Hiện trạng kiến trúc & Bottlenecks
- **Kiến trúc cũ:** Sử dụng Naive RAG cơ bản: Chia đoạn văn bản bằng `RecursiveCharacterTextSplitter` (chunk size 1000, overlap 100), embedding bằng OpenAI `text-embedding-3-small`, lưu trữ trong FAISS/Chroma, truy vấn dense-only và sinh câu trả lời trực tiếp không qua reranking.
- **Điểm nghẽn thực tế:**
  1. *Version Conflict:* Khi công ty ban hành quy định mới (ví dụ chính sách nghỉ phép 2024 thay cho 2023), mô hình dense thường truy xuất nhầm văn bản cũ do độ tương đồng ngữ nghĩa về mặt từ ngữ giữa 2 văn bản gần như tương đương.
  2. *Low Context Precision:* Các câu hỏi về số liệu chính xác (mức công tác phí, số ngày thử việc, thời hạn nộp giấy tờ ốm đau) thường bị trôi mất thông tin quan trọng vì chunk 1000 ký tự chứa quá nhiều thông tin rác.
  3. *Thiếu cơ chế Benchmark chuẩn:* Không đo lường được mức độ hallucination định lượng của bot trước khi release bản cập nhật.

#### 2. Kế hoạch cải tiến cụ thể
1. **Chiến lược Chunking:**
   - Triển khai **Hierarchical Chunking (Parent-Child)** làm chiến lược cốt lõi: Child chunk kích thước 256 ký tự để bắt trúng các câu/điều khoản cụ thể; Parent chunk kích thước 1500–2048 ký tự chứa ngữ cảnh toàn bộ chương/mục nhằm cung cấp cho LLM sinh câu trả lời đầy đủ.
   - Đối với tài liệu có format Markdown / Hướng dẫn công việc có đề mục rõ ràng: Sử dụng **Structure-Aware Chunking** để bảo toàn bảng biểu quy định trợ cấp và tiêu đề chương mục.
2. **Công cụ Tìm kiếm (Search & Retrieval):**
   - Chuyển đổi sang **Hybrid Search**: Kết hợp **BM25 Tiếng Việt** (sử dụng `underthesea` segmentation) và **Dense Retrieval** (`BAAI/bge-m3` hoặc `text-embedding-3-large`).
   - Hợp nhất kết quả bằng **Reciprocal Rank Fusion (RRF)** với $k=60$. BM25 sẽ giải quyết dứt điểm các truy vấn chứa từ khóa kỹ thuật, số hiệu văn bản (Nghị định 13, Form BM-05...), còn Dense bao quát các câu hỏi diễn đạt tự nhiên của nhân viên.
3. **Bộ Tái xếp hạng (Reranking):**
   - Tích hợp mô hình `BAAI/bge-reranker-v2-m3` để rerank từ Top 20 kết quả của Hybrid Search xuống Top 3–5 chunks chất lượng nhất trước khi truyền vào prompt context. Thêm bộ lọc metadata `is_active=True` để loại bỏ phiên bản chính sách cũ đã hết hiệu lực.
4. **Enrichment Pipeline:**
   - Áp dụng **Contextual Prepend** kết hợp kỹ thuật **Combined Single-Call**: Mỗi chunk tài liệu trước khi index sẽ được bổ sung 1 câu bối cảnh vị trí tài liệu (ví dụ: *"Trích từ Sổ tay Nhân viên 2024, Chương 3: Chế độ Nghỉ phép"*). Kỹ thuật này theo nghiên cứu của Anthropic giúp giảm đến 40–50% lỗi truy xuất trượt.
5. **Hệ thống Đo lường (Evaluation & LLM Observability):**
   - Tự động hóa đánh giá định kỳ trên bộ testset chuẩn hóa (Golden Dataset gồm 50+ Q&A đa dạng thể loại: Lookup, Numeric, Version Conflict, Negation).
   - Thiết lập ngưỡng kiểm duyệt chất lượng CI/CD: Bắt buộc chỉ số **Faithfulness $\ge 0.85$** và **Context Recall $\ge 0.80$** qua RAGAS mới được phép deploy lên môi trường Production.

#### 3. Timeline triển khai (4 tuần)
- **Tuần 1: Refactor Ingestion & Chunking**
  - Chuyển đổi toàn bộ kho tài liệu PDF/Word/Markdown sang dạng cấu trúc.
  - Implement pipeline phân đoạn Hierarchical và Structure-Aware Chunking.
  - Tích hợp bước tiền xử lý Contextual Prepend & Auto Metadata Extraction.
- **Tuần 2: Xây dựng Hybrid Search Engine & Reranker**
  - Cài đặt Qdrant cluster trên hạ tầng nội bộ.
  - Cấu hình BM25 Tiếng Việt với bộ từ điển chuyên ngành doanh nghiệp.
  - Triển khai RRF fusion và tích hợp microservice Cross-Encoder Reranker (`bge-reranker-v2-m3`).
- **Tuần 3: Benchmark RAGAS & Prompt Optimization**
  - Xây dựng bộ Test Set gồm 60 câu hỏi thuộc các bộ phận HR, IT, Kế toán, Pháp chế.
  - Chạy RAGAS benchmark, vẽ biểu đồ so sánh Baseline vs Hybrid Rerank.
  - Tinh chỉnh Prompt template, bổ sung guardrails xử lý khi không tìm thấy thông tin nhằm đạt Faithfulness $\ge 0.90$.
- **Tuần 4: User Acceptance Testing (UAT) & Monitoring**
  - Đóng gói Docker Compose / Kubernetes triển khai thử nghiệm cho 50 nhân viên nội bộ.
  - Thiết lập logging theo dõi Latency, Top-k context, và tỷ lệ phản hồi hữu ích (Thumb Up / Down).
