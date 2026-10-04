# Failure Analysis — Lab 18: Production RAG

**Họ và tên học viên:** Lê Như Ý  
**MSSV:** 202602517  
**Khóa:** K4 - Track 3B  

---

## RAGAS Scores

| Metric | Naive Baseline | Production | Δ |
|--------|---------------|------------|---|
| Faithfulness | 0.8495 | 0.7583 | -0.0912 |
| Answer Relevancy | 0.7210 | 0.6440 | -0.0770 |
| Context Precision | 0.9226 | 0.8917 | -0.0309 |
| Context Recall | 0.9375 | 0.8167 | -0.1208 |

## Bottom-5 Failures

### #1
- **Question:** Thông tin lương thuộc cấp độ phân loại dữ liệu nào?
- **Expected:** Theo quy chế chi trả lương, thông tin lương được phân loại là dữ liệu Bí mật, cấm chia sẻ với đồng nghiệp. Theo chính sách phân loại dữ liệu, dữ liệu Bí mật (cấp 3) phải mã hóa khi truyền và hạn chế truy cập theo need-to-know.
- **Got:** Thông tin lương thuộc cấp độ dữ liệu Bí mật (cấm chia sẻ với đồng nghiệp).
- **Worst metric:** Faithfulness (Score: 0.0)
- **Error Tree:** Output đúng thông tin cốt lõi nhưng bị chấm Faithfulness = 0.0 → Context retrieval chỉ lấy được văn bản Quy chế lương mà thiếu văn bản Phân loại dữ liệu cấp 3 → LLM không trích dẫn đủ 2 văn bản đồng thời.
- **Root cause:** Context retrieval bị phân mảnh giữa 2 tài liệu độc lập (`Quy chế chi trả lương` và `Chính sách phân loại dữ liệu`). RAGAS Faithfulness evaluator kiểm tra sự xuất hiện của các phát biểu về "mã hóa khi truyền và hạn chế need-to-know" không có trong context được nạp nên đánh phạt toàn bộ câu trả lời.
- **Suggested fix:** Thắt chặt system prompt ("Chỉ trả lời dựa trên context, nếu context thiếu một phần thông tin hãy nêu rõ"), bổ sung kỹ thuật Cross-Document Linking ở khâu Chunking/Enrichment để liên kết các điều khoản liên quan giữa các quy chế.

### #2
- **Question:** Muốn mua thiết bị trị giá 55 triệu cần ai phê duyệt?
- **Expected:** Đơn hàng trên 50.000.000 VNĐ cần Tổng Giám đốc (CEO) phê duyệt.
- **Got:** Mua thiết bị trị giá 55 triệu VNĐ (trên 50 triệu) cần Tổng Giám đốc (CEO) phê duyệt.
- **Worst metric:** Context Precision (Score: 0.0)
- **Error Tree:** Output đúng → Context chứa thông tin đúng nhưng nằm ở rank thấp → Top rank bị chiếm bởi các chunks hạn mức khác (5-50 triệu, dưới 5 triệu).
- **Root cause:** Trong Top 3 chunks trả về sau Reranking, các chunks về quy trình mua sắm chung và hạn mức phê duyệt của Trưởng phòng/Giám đốc (dưới 50 triệu) có điểm BM25 lexical match rất cao với từ khóa "mua thiết bị", đẩy chunk chứa quy tắc đặc biệt "> 50 triệu: Tổng Giám đốc" xuống vị trí sau hoặc ngoài Top precision.
- **Suggested fix:** Áp dụng Metadata Filtering theo phạm vi ngân sách (`budget_threshold >= 50M`), tăng trọng số reranker cho các điều khoản có điều kiện số học và loại trừ các chunk không khớp khoảng giá trị.

### #3
- **Question:** Bao lâu phải đổi mật khẩu một lần?
- **Expected:** Theo chính sách hiện hành (v2.0), mật khẩu phải được thay đổi mỗi 120 ngày. Chính sách cũ yêu cầu 90 ngày nhưng đã bị thay thế.
- **Got:** Theo chính sách quản lý mật khẩu v2.0 hiện hành, mật khẩu phải được thay đổi định kỳ mỗi 120 ngày.
- **Worst metric:** Faithfulness (Score: 0.0)
- **Error Tree:** Output đúng thực tế chính sách hiện hành → Context chứa cả văn bản cũ (v1.0: 90 ngày) và văn bản mới (v2.0: 120 ngày) → Mâu thuẫn phiên bản (Version Conflict).
- **Root cause:** Tài liệu trong kho dữ liệu chứa cả văn bản cũ chưa bị xóa hoàn toàn. RAGAS LLM-judge nhận thấy context chứa thông tin mâu thuẫn ("90 ngày" và "120 ngày"), kết luận câu trả lời khẳng định 120 ngày mà không giải thích sự đối nghịch với chunk 90 ngày là thiếu bám sát context tổng thể.
- **Suggested fix:** Thêm bước Metadata Enrichment đánh dấu trường `status: "deprecated"` cho văn bản v1.0 và lọc bỏ tài liệu cũ trước khi Indexing; cập nhật prompt sinh câu trả lời đối chiếu rõ ràng: "Theo chính sách v2.0 mới nhất thay thế v1.0...".

### #4
- **Question:** Lương thử việc của nhân viên Junior mức cao nhất là bao nhiêu?
- **Expected:** Junior cao nhất là 20.000.000 VNĐ/tháng. Lương thử việc = 85% x 20.000.000 = 17.000.000 VNĐ/tháng.
- **Got:** Lương thử việc cao nhất của nhân viên Junior là 17.000.000 VNĐ/tháng (bằng 85% của mức trần Junior 20.000.000 VNĐ/tháng).
- **Worst metric:** Faithfulness (Score: 0.0)
- **Error Tree:** Output đúng logic tính toán → Context chỉ chứa hai tiền đề tách rời ("Junior: 12-20tr" và "Lương thử việc: 85%") → Con số 17.000.000 không xuất hiện nguyên văn trong context chuỗi văn bản.
- **Root cause:** RAGAS Faithfulness sử dụng NLI (Natural Language Inference) chặt chẽ: khi con số 17.000.000 VNĐ không có mặt nguyên văn (verbatim) trong context mà do LLM thực hiện phép tính $20 \times 85\%$, bộ chấm điểm suy đoán đó là hallucination/unsupported claim.
- **Suggested fix:** Yêu cầu prompt trả lời theo định dạng Chain-of-Thought nêu rõ: "1) Căn cứ Điều X: lương Junior tối đa 20tr; 2) Căn cứ Điều Y: thử việc hưởng 85%; 3) Suy luận: 85% × 20tr = 17tr" để LLM-judge phân biệt giữa suy luận logic và ảo giác.

### #5
- **Question:** Một nhân viên Senior có 9 năm thâm niên được nghỉ bao nhiêu ngày phép năm và lương trong khoảng nào?
- **Expected:** Theo chính sách v2024: 15 ngày cơ bản + 3 ngày thâm niên (9÷3=3) = 18 ngày phép. Lương Senior (P3-P4): 20-35 triệu VNĐ/tháng.
- **Got:** Nhân viên Senior có 9 năm thâm niên được nghỉ 18 ngày phép năm theo chính sách v2024 (15 ngày cơ bản + 3 ngày thâm niên). Khung lương Senior nằm trong khoảng 20 - 35 triệu VNĐ/tháng.
- **Worst metric:** Answer Relevancy (Score: 0.0)
- **Error Tree:** Output trả lời đầy đủ cả 2 vế → Câu hỏi phức hợp 2 mệnh đề → Đánh giá ngữ nghĩa cosine embedding giữa câu hỏi kép và câu trả lời diễn giải dài.
- **Root cause:** RAGAS Answer Relevancy sinh các câu hỏi nhân tạo từ Answer rồi đo độ tương đồng với câu hỏi gốc. Do câu trả lời chứa nhiều số liệu kỹ thuật về điều kiện thâm niên và bảng lương P3-P4, các câu hỏi đảo ngược sinh ra bị phân tán chủ đề (một số chỉ hỏi về thâm niên, một số chỉ hỏi về lương), làm giảm điểm tương đồng ngữ nghĩa.
- **Suggested fix:** Cải thiện cấu trúc prompt yêu cầu định dạng câu trả lời ngắn gọn, trực diện, phân tách rõ 2 gạch đầu dòng tương ứng với 2 vế của câu hỏi để câu hỏi đảo ngược khớp chính xác với câu hỏi gốc.

## Case Study (cho presentation)

**Question chọn phân tích:**  
`"Một nhân viên Senior có 9 năm thâm niên được nghỉ bao nhiêu ngày phép năm và lương trong khoảng nào?"`

**Expected Ground Truth:**  
Theo chính sách v2024: 15 ngày cơ bản + 3 ngày thâm niên (9 ÷ 3 = 3) = 18 ngày phép. Lương Senior (P3-P4): 20-35 triệu VNĐ/tháng.

**Error Tree walkthrough:**
1. **Output đúng?**  
   → Trong mô hình Naive Baseline, câu trả lời thường bị thiếu một trong hai vế (hoặc tính nhầm số ngày phép thành 12 + 1 = 13 ngày do bốc phải quy định v2023 cũ "mỗi 5 năm thâm niên tăng 1 ngày", hoặc hoàn toàn bỏ sót khung lương P3-P4).
2. **Context đúng?**  
   → Context retrieval ở Naive Dense-only gặp hiện tượng "version mismatch" và "document dispersion": Chunks chứa quy định nghỉ phép nằm ở file quy định nhân sự v2024, còn chunks chứa bảng lương Senior nằm ở file cấu trúc đãi ngộ. Dense Search đơn thuần với Top 3 chỉ ưu tiên các chunks có cosine similarity cao nhất với từ khóa "phép năm" mà bỏ rơi bảng lương, khiến LLM bị thiếu context để suy luận đầy đủ.
3. **Query rewrite OK?**  
   → Câu hỏi của người dùng là câu hỏi đa mục tiêu (multi-hop / composite query) kết hợp 2 thực thể độc lập: [Chính sách phép thâm niên] và [Khung lương chức danh]. Nếu không thực hiện query decomposition, BM25 và Vector Search bị phân tán trọng số tìm kiếm.
4. **Fix ở bước:**  
   - **M1 + M5 (Chunking & Enrichment):** Hierarchical chunking bảo toàn quan hệ cha-con và M5 Contextual Prepend bổ sung rõ nguồn tài liệu `"Chính sách nhân sự v2024 (thay thế v2023)"`, giúp triệt tiêu nhầm lẫn phiên bản.
   - **M2 + M3 (Hybrid Search + Cross-Encoder Rerank):** Mở rộng retrieval lên Top 20 từ cả BM25 (bắt chính xác token "Senior", "9 năm", "thâm niên") và Dense (bắt ngữ nghĩa "nghỉ phép", "lương"), sau đó qua Cross-Encoder lọc chuẩn Top 3 chứa đầy đủ cả 2 khía cạnh thông tin.

---

## Nếu có thêm 1 giờ, sẽ optimize:

1. **Query Decomposition & HyDE (Sub-query Routing):**
   - Triển khai bước tiền xử lý truy vấn: Tự động phát hiện các câu hỏi phức hợp (chứa liên từ "và", nhiều yêu cầu) để phân rã thành các truy vấn đơn nguyên (atomic queries).
   - Áp dụng Hypothetical Document Embeddings (HyDE) cho các câu hỏi mang tính suy luận hoặc chính sách đặc thù.

2. **Temporal & Version Metadata Filtering (Metadata Router):**
   - Thêm quy tắc lọc cứng (hard filter) theo metadata `status: "active"` hoặc `effective_date: "2024"` được trích xuất từ M5 để ngăn chặn 100% rủi ro truy xuất các tài liệu đã hết hiệu lực (v2023, v1.0).

3. **Dynamic Score Fusion & Adaptive Retrieval Thresholds:**
   - Thay vì cố định $k = 60$ trong RRF, tối ưu trọng số kết hợp động giữa BM25 và Dense dựa trên đặc tính câu hỏi (câu hỏi chứa mã hiệu/con số cụ thể ưu tiên BM25; câu hỏi hỏi về định nghĩa/khái niệm ưu tiên Dense).
   - Tích hợp Self-Correction / Corrective RAG (CRAG) để chấm điểm độ tự tin của context trước khi gọi LLM sinh đáp án.

