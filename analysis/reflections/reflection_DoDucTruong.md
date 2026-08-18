# Individual Reflection — Lab 18: Production RAG Pipeline

**Họ và tên:** Đỗ Đức Trường  
**Mã sinh viên:** 2A202601499  
**Lớp:** AICB-K34 · Ngày 18: Production RAG  
**Phụ trách:** Toàn bộ 5 Modules (M1 Chunking, M2 Hybrid Search, M3 Reranking, M4 Evaluation, M5 Enrichment)

---

## Phần 1: Mapping bài giảng vào Code thực tế (Lecture Mapping)

| Khái niệm bài giảng (Lecture Concept) | Module | Hàm / Class cụ thể | Quan sát & Nhận định thực tế (Observation) |
|---|---|---|---|
| **Semantic Chunking** | M1 | [`chunk_semantic()`](file:///Users/doductruong.workgmail.com/Documents/ai%20lab/K34-Day18-2A202601499-DoDucTruong/src/m1_chunking.py#L84-L106) | Sử dụng `SentenceTransformer("all-MiniLM-L6-v2")` để tính cosine similarity giữa các câu liền kề. Với ngưỡng threshold = 0.85, thuật toán tự động gom các câu cùng mạch ý nghĩa thành một chunk, tránh việc cắt đứt câu giữa chừng như basic paragraph chunking. |
| **Hierarchical Chunking (Parent-Child)** | M1 | [`chunk_hierarchical()`](file:///Users/doductruong.workgmail.com/Documents/ai%20lab/K34-Day18-2A202601499-DoDucTruong/src/m1_chunking.py#L109-L130) | Tách tài liệu thành Parent chunks (2048 chars) và Child chunks (256 chars). Khi truy vấn, hệ thống tìm kiếm trên các child chunks để đạt precision cao, nhưng trả về parent chunk cho LLM để cung cấp đầy đủ ngữ cảnh rộng (context recall). |
| **Structure-Aware Chunking** | M1 | [`chunk_structure_aware()`](file:///Users/doductruong.workgmail.com/Documents/ai%20lab/K34-Day18-2A202601499-DoDucTruong/src/m1_chunking.py#L134-L148) | Phân tách tài liệu theo tiêu đề Markdown (`#`, `##`, `###`), gắn metadata `section` vào từng chunk. Giữ nguyên vẹn bảng biểu (tables) và danh sách (lists) trong cùng một section. |
| **BM25 + Dense Fusion (Hybrid Search)** | M2 | [`reciprocal_rank_fusion()`](file:///Users/doductruong.workgmail.com/Documents/ai%20lab/K34-Day18-2A202601499-DoDucTruong/src/m2_search.py#L98-L111) & [`HybridSearch`](file:///Users/doductruong.workgmail.com/Documents/ai%20lab/K34-Day18-2A202601499-DoDucTruong/src/m2_search.py#L113-L127) | Kết hợp BM25 (từ khóa chính xác tiếng Việt sau khi phân từ bằng `underthesea`) và Dense Vector (`BAAI/bge-m3` trên Qdrant). RRF với công thức $Score(d) = \sum \frac{1}{k + rank + 1}$ giải quyết triệt để sự khác biệt về phân phối điểm số (score distribution) giữa vector cosine và BM25 score. |
| **Cross-Encoder Reranking** | M3 | [`CrossEncoderReranker.rerank()`](file:///Users/doductruong.workgmail.com/Documents/ai%20lab/K34-Day18-2A202601499-DoDucTruong/src/m3_rerank.py#L21-L50) | Mô hình `BAAI/bge-reranker-v2-m3` tiếp nhận toàn bộ cặp `(Query, Passage)` đồng thời, tính toán cross-attention toàn diện giữa câu hỏi và tài liệu. Giúp lọc từ Top 20 ứng viên xuống Top 3 ngữ cảnh chính xác nhất, triệt tiêu tài liệu nhiễu trước khi đưa vào LLM context window. |
| **RAGAS 4 Metrics Evaluation** | M4 | [`evaluate_ragas()`](file:///Users/doductruong.workgmail.com/Documents/ai%20lab/K34-Day18-2A202601499-DoDucTruong/src/m4_eval.py#L30-L62) & [`failure_analysis()`](file:///Users/doductruong.workgmail.com/Documents/ai%20lab/K34-Day18-2A202601499-DoDucTruong/src/m4_eval.py#L64-L77) | Đo lường hệ thống qua 4 trục: Faithfulness (độ trung thực, chống ảo giác), Answer Relevancy (độ phù hợp của câu trả lời), Context Precision (tỉ lệ chunk liên quan ở vị trí đầu) và Context Recall (độ phủ thông tin). Áp dụng Diagnostic Tree để phân loại lỗi có hệ thống. |
| **Chunk Enrichment (Contextual Embeddings)** | M5 | [`_enrich_single_call()`](file:///Users/doductruong.workgmail.com/Documents/ai%20lab/K34-Day18-2A202601499-DoDucTruong/src/m5_enrichment.py#L157-L186) & [`enrich_chunks()`](file:///Users/doductruong.workgmail.com/Documents/ai%20lab/K34-Day18-2A202601499-DoDucTruong/src/m5_enrichment.py#L191-L244) | Bổ sung ngữ cảnh xuất xứ tài liệu (Contextual Prepend phong cách Anthropic), sinh câu hỏi giả định (HyQA) để thu hẹp khoảng cách từ vựng (vocabulary gap), tạo tóm tắt và trích xuất metadata trước khi index. Tối ưu hóa chi phí với chế độ combined (1 API call/chunk). |

---

## Phần 2: Khó khăn gặp phải & Cách giải quyết (Challenges & Debugging)

1. **Khắc phục xung đột phân từ tiếng Việt với BM25:**
   - *Lỗi gặp phải:* Khi dùng `underthesea.word_tokenize(text, format="text")`, các từ ghép tiếng Việt được nối bằng dấu gạch dưới (ví dụ `"nghỉ_phép"`). Tuy nhiên, thư viện BM25 (`rank_bm25`) mặc định tokenize theo khoảng trắng (`split()`), khiến query `"nghỉ phép"` (2 tokens) không khớp được với chunk chứa `"nghỉ_phép"` (1 token).
   - *Cách giải quyết:* Viết hàm [`segment_vietnamese()`](file:///Users/doductruong.workgmail.com/Documents/ai%20lab/K34-Day18-2A202601499-DoDucTruong/src/m2_search.py#L21-L33) thực hiện `word_tokenize(text, format="text").replace("_", " ")`, giúp BM25 nhận diện chuẩn xác các từ vựng tiếng Việt.

2. **Tương thích Qdrant Client API (v1.9+ / v2.0):**
   - *Lỗi gặp phải:* Gọi `client.search()` bị cảnh báo deprecation hoặc sai signature trong các phiên bản mới của `qdrant-client`.
   - *Cách giải quyết:* Chuyển sang sử dụng `client.query_points(collection_name=..., query=..., limit=...)` theo đúng chuẩn API hiện đại.

3. **Tối ưu chi phí và độ trễ ở Module Enrichment (M5):**
   - *Lỗi gặp phải:* Nếu gọi riêng lẻ 4 tác vụ (summarize, HyQA, contextual prepend, auto metadata) sẽ tốn 4 API calls cho mỗi chunk, gây chậm pipeline và dễ chạm rate limit khi corpus lớn.
   - *Cách giải quyết:* Thiết kế prompt hợp nhất trong [`_enrich_single_call()`](file:///Users/doductruong.workgmail.com/Documents/ai%20lab/K34-Day18-2A202601499-DoDucTruong/src/m5_enrichment.py#L157-L186) yêu cầu LLM trả về cấu trúc JSON duy nhất chứa đầy đủ summary, questions, context line và metadata trong 1 request duy nhất (giảm 75% API calls).

---

## Phần 3: Kế hoạch áp dụng vào Dự án Cá nhân (Action Plan)

### Dự án: Hệ thống Trợ lý Hỏi đáp Quy định & Tài liệu Kỹ thuật Nội bộ (Enterprise Internal KB Assistant)

### 1. Hiện trạng (Current State)
- Đang sử dụng Naive RAG cơ bản (cắt chunk cố định 500 ký tự, chỉ tìm kiếm vector cosine với OpenAI embeddings).
- **Vấn đề tồn đọng:** 
  - Các câu hỏi liên quan đến bảng biểu (chính sách lương, quy định tài chính) thường bị cắt rời làm mất hàng tiêu đề.
  - Các câu hỏi có từ khóa tiếng Việt chính xác (mã quy định, tên viết tắt, số nghị định) đôi khi không tìm được do embedding bị phân tán.
  - Gặp hiện tượng version conflict (văn bản cũ và văn bản mới cùng tồn tại dẫn đến trả lời sai phiên bản hiệu lực).

### 2. Kế hoạch cải tiến với Production RAG Pipeline

| Thành phần | Chiến lược lựa chọn | Lý do & Rationale |
|---|---|---|
| **1. Chunking Strategy** | **Hierarchical Chunking (Parent-Child)** kết hợp **Structure-Aware** cho Markdown docs | Giúp bảo toàn cấu trúc bảng biểu và phân mục; khi tìm kiếm dùng child chunk để đạt precision, khi sinh câu trả lời dùng parent chunk để LLM có ngữ cảnh hoàn chỉnh. |
| **2. Search Strategy** | **Hybrid Search (BM25 + BGE-M3 + RRF)** | BM25 bắt chính xác các mã số văn bản, từ khóa kỹ thuật tiếng Việt; BGE-M3 bắt tốt ngữ nghĩa; RRF dung hợp cân bằng cả hai nguồn mà không cần chuẩn hóa scale điểm số phức tạp. |
| **3. Reranking** | **Cross-Encoder `BAAI/bge-reranker-v2-m3`** (hoặc `Flashrank` cho microservice độ trễ thấp) | Lọc top 20 retrieval xuống top 3-5 passages thực sự liên quan nhất, giảm tải token context và triệt tiêu thông tin gây nhiễu cho generator. |
| **4. Evaluation** | **RAGAS Framework (4 metrics)** + Custom Benchmark Testset 50 câu hỏi | Đánh giá định lượng thường xuyên trước và sau mỗi lần cập nhật corpus dữ liệu. |
| **5. Enrichment** | **Contextual Prepend (Anthropic style) + Auto Metadata (Version, Effective Date)** | Giúp giải quyết triệt để bài toán Version Conflict bằng cách bổ sung metadata phiên bản (`v2024`, `active: true`) vào từng chunk để filter. |

### 3. Lộ trình triển khai (Timeline)
- **Tuần 1:** Tái cấu trúc pipeline tiền xử lý dữ liệu: Triển khai Markdown parsing + Hierarchical Chunking + Contextual Prepend.
- **Tuần 2:** Thiết lập cụm Qdrant kết hợp BM25Okapi tiếng Việt, cấu hình hàm RRF dung hợp kết quả.
- **Tuần 3:** Tích hợp Cross-Encoder Reranker, tinh chỉnh prompt tạo câu trả lời của LLM với quy tắc trích dẫn nguồn (source citations).
- **Tuần 4:** Xây dựng bộ testset đánh giá tự động bằng RAGAS, chạy regression test và benchmark độ trễ (latency profiling).
