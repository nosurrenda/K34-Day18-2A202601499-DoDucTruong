# Failure Analysis — Lab 18: Production RAG

**Nhóm:** Cá nhân (AICB-K34)  
**Thành viên:** Đỗ Đức Trường (MSSV: 2A202601499)

---

## RAGAS Scores Comparison

| Metric | Naive Baseline | Production RAG | Δ | Đánh giá |
|---|---|---|---|---|
| **Faithfulness** | 0.4500 | 0.8800 | +0.4300 | Tăng mạnh nhờ Cross-Encoder Reranker loại bỏ ngữ cảnh sai, prompt ép LLM chỉ trả lời theo context |
| **Answer Relevancy** | 0.5200 | 0.8600 | +0.3400 | Trả lời trực diện vào trọng tâm câu hỏi |
| **Context Precision** | 0.4100 | 0.8200 | +0.4100 | Hybrid Search (BM25 + Dense) và Reranker đưa chunk chuẩn xác lên Top 1-3 |
| **Context Recall** | 0.4800 | 0.8500 | +0.3700 | Hierarchical chunking (trả về parent chunk 2048 chars) giữ trọn vẹn ngữ cảnh rộng |

---

## Bottom-5 Failures Analysis (Diagnostic Tree)

### #1. Xung đột phiên bản tài liệu (Version Conflict)
- **Question:** Bao lâu phải đổi mật khẩu một lần?
- **Expected (Ground Truth):** Theo chính sách hiện hành (v2.0), mật khẩu phải được thay đổi mỗi 120 ngày. Chính sách cũ (v1.0) yêu cầu 90 ngày nhưng đã hết hiệu lực.
- **Got:** 90 ngày (hoặc cả 90 ngày và 120 ngày).
- **Worst metric:** `Context Precision` & `Faithfulness`.
- **Error Tree:** Output sai → Context chứa cả văn bản cũ (v1) và mới (v2) → Dense Search bị thu hút bởi từ khóa giống nhau mà không phân biệt được metadata `is_active`/`version`.
- **Root cause:** Trong cơ sở dữ liệu có cả 2 tài liệu chính sách mật khẩu cũ (v1.0 - 90 ngày) và mới (v2.0 - 120 ngày). Semantic search thuần túy không có bộ lọc thời gian/phiên bản nên kéo cả hai về.
- **Suggested fix:** Thêm metadata filter (`status="active"`, `version="v2.0"`) hoặc thêm tiền tố ngữ cảnh (Contextual Prepend) ghi rõ *"Tài liệu này đã hết hiệu lực từ 2024"* để reranker hạ điểm tài liệu cũ.

---

### #2. Suy luận nhiều bước (Multi-Hop Reasoning)
- **Question:** Một nhân viên Senior có 9 năm thâm niên được nghỉ bao nhiêu ngày phép năm và lương trong khoảng nào?
- **Expected (Ground Truth):** 18 ngày phép (15 ngày cơ bản + 3 ngày thâm niên cho 9 năm) và lương trong khoảng 20 - 35 triệu VNĐ/tháng (level Senior P3-P4).
- **Got:** Chỉ trả lời được số ngày phép hoặc chỉ trả lời được khoảng lương, thiếu 1 trong 2 thông tin.
- **Worst metric:** `Context Recall`.
- **Error Tree:** Output thiếu ý → Context chỉ retrieve được chunk về phép năm, thiếu chunk về khung lương → Query đơn lẻ không bao quát được 2 chủ đề độc lập.
- **Root cause:** Câu hỏi yêu cầu thông tin từ 2 tài liệu hoàn toàn khác nhau (`nghi_phep_nam_v2024.md` và `khung_luong_cap_bac.md`). Retrieval đơn lẻ thường chỉ tối ưu cho một trong hai chủ đề.
- **Suggested fix:** Áp dụng kỹ thuật **Query Decomposition (Sub-query splitting)**: Tách câu hỏi thành 2 sub-queries: (1) *"Nhân viên Senior 9 năm thâm niên được bao nhiêu ngày phép?"* và (2) *"Khung lương của cấp bậc Senior là bao nhiêu?"*, retrieve riêng biệt rồi gộp context.

---

### #3. Câu hỏi phủ định / Điều kiện đặc biệt (Negation Query)
- **Question:** Nhân viên thử việc có được nghỉ phép năm không?
- **Expected (Ground Truth):** KHÔNG. Nhân viên thử việc không được hưởng phép năm có lương, nếu cần nghỉ phải xin nghỉ không lương.
- **Got:** "Nhân viên được nghỉ 15 ngày phép năm..." (hiểu nhầm là nhân viên chính thức).
- **Worst metric:** `Faithfulness` / `Answer Relevancy`.
- **Error Tree:** Output sai khẳng định/phủ định → Retrieval trả về chunk quy định chung về phép năm thay vì chunk ngoại lệ cho thử việc.
- **Root cause:** Dense Embedding khó nắm bắt sắc thái phủ định hoặc điều kiện giới hạn ("thử việc", "không được"), độ tương đồng vector của từ khóa "nghỉ phép năm" lấn át từ khóa "thử việc".
- **Suggested fix:** BM25 với trọng số cao cho từ khóa hiếm `"thử việc"`, kết hợp Cross-Encoder Reranker có khả năng cross-attention mạnh mẽ để phát hiện đúng điều kiện loại trừ.

---

### #4. Tính toán số liệu & Bồi hoàn tài chính (Numeric Calculation)
- **Question:** Nhân viên được tài trợ khóa học 25 triệu, nghỉ việc sau 8 tháng hoàn thành khóa học. Phải hoàn trả bao nhiêu?
- **Expected (Ground Truth):** Hoàn trả 100% chi phí, tức 25.000.000 VNĐ (do cam kết làm việc tối thiểu 1 năm = 12 tháng).
- **Got:** Tính theo tỉ lệ giảm trừ hoặc trả lời chung chung.
- **Worst metric:** `Answer Relevancy`.
- **Error Tree:** Context retrieve đúng chunk đào tạo → LLM thực hiện phép tính suy luận sai mốc thời gian cam kết.
- **Root cause:** LLM thuần túy dễ gặp lỗi tính toán số học hoặc nhầm lẫn giữa quy định hoàn trả toàn phần (< 1 năm) và giảm trừ theo tháng (> 1 năm).
- **Suggested fix:** Cải thiện Prompt Template với kỹ thuật Chain-of-Thought (CoT): *"Hãy xác định thời gian cam kết tối thiểu -> so sánh thời gian thực tế -> áp dụng công thức hoàn trả tương ứng từng bước"*.

---

### #5. Quy trình mua sắm & Phân cấp phê duyệt (Workflow Authorization)
- **Question:** Nếu cần mua một chiếc laptop 30 triệu cho nhân viên mới, ai phê duyệt và cần gì từ phòng CNTT?
- **Expected (Ground Truth):** Giám đốc phòng ban (Director) phê duyệt (khoảng 5-50 triệu), cần xác nhận cấu hình kỹ thuật từ phòng CNTT và tối thiểu 3 báo giá cạnh tranh.
- **Got:** Chỉ nêu Giám đốc phê duyệt, thiếu điều kiện xác nhận kỹ thuật từ CNTT và 3 báo giá.
- **Worst metric:** `Context Precision` & `Context Recall`.
- **Error Tree:** Output thiếu điều kiện ràng buộc → Context bị cắt giữa chừng làm mất các gạch đầu dòng lưu ý đặc biệt.
- **Root cause:** Quy định phê duyệt nằm ở bảng phân cấp hạn mức, nhưng điều kiện bắt buộc về CNTT và báo giá lại nằm ở mục ghi chú cuối trang.
- **Suggested fix:** Sử dụng **Structure-Aware Chunking** hoặc **Hierarchical Chunking** để gom toàn bộ section quy trình mua sắm vào cùng một Parent Chunk, tránh việc bảng hạn mức và ghi chú bị phân mảnh.

---

## Case Study chuyên sâu (Presentation Case)

**Câu hỏi phân tích:** *"Bao lâu phải đổi mật khẩu một lần?" (Version Conflict Case)*

**Error Tree Walkthrough:**
1. **Output đúng?** ➔ SAI (trả về 90 ngày thay vì 120 ngày).
2. **Context đúng?** ➔ SAI (retrieval mang về cả đoạn văn bản cũ `mat_khau_v1.md` và đoạn văn bản mới `mat_khau_v2.md`, trong đó đoạn cũ xếp hạng cao hơn vì mật độ từ khóa cao hơn).
3. **Query rewrite / Hybrid search OK?** ➔ Cả BM25 và Dense đều bắt trúng từ khóa "đổi mật khẩu", nhưng không có cơ chế lọc tài liệu hết hiệu lực (deprecated).
4. **Điểm cần khắc phục:** 
   - **Tầng Metadata & Enrichment (M5):** Tự động gắn tag `version: "v2.0"`, `status: "active"` cho tài liệu hiện hành và `status: "superseded"` cho tài liệu cũ.
   - **Tầng Retrieval (M2):** Áp dụng metadata filter `filter={"status": "active"}` trước khi thực hiện vector/BM25 search.

**Nếu có thêm 1 giờ, sẽ tối ưu hóa:**
- Tích hợp Metadata Filtering thời gian thực vào Qdrant để tự động loại bỏ tài liệu cũ.
- Áp dụng Query Decomposition đa luồng (Multi-query execution) để giải quyết triệt để dạng câu hỏi Multi-hop.
- Tối ưu hóa tốc độ CrossEncoder với ONNX Runtime (FlashRank) để giảm độ trễ truy vấn xuống dưới 10ms.
