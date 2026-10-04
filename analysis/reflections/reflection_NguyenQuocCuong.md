# Individual Reflection — Lab 18: Production RAG

**Họ và tên:** Nguyễn Quốc Cường  
**Khóa:** K4 - Track 3B  
**Ngày hoàn thành:** 05/10/2026

---

## Phần 1: Mapping bài giảng (Lecture Mapping)

| Khái niệm | Module và hàm | Điều đã triển khai và quan sát |
|---|---|---|
| Semantic chunking | M1 — `chunk_semantic()` | Tách câu, mã hóa bằng `all-MiniLM-L6-v2`, ngắt khi cosine similarity dưới 0,85. Chiến lược này đã viết nhưng pipeline chính dùng hierarchical chunking; chưa có số liệu so sánh riêng. |
| Parent–child và structure-aware chunking | M1 — `chunk_hierarchical()`, `chunk_structure_aware()` | Đoạn cha tối đa 2048 ký tự, đoạn con tối đa 256 ký tự mang `parent_id`. Pipeline hiện chỉ gửi văn bản child vào LLM; chưa truy ngược parent, nên có ca mất từ phủ định hoặc hàng bảng. |
| Hybrid retrieval và RRF | M2 — `BM25Search`, `DenseSearch`, `reciprocal_rank_fusion()` | BM25 xử lý từ khóa chính xác, Dense dùng `BAAI/bge-m3` và Qdrant; RRF cộng `1/(60 + rank + 1)` theo thứ hạng. Tách từ tiếng Việt rồi đổi `_` thành khoảng trắng. |
| Cross-encoder reranking | M3 — `CrossEncoderReranker.rerank()` | `BAAI/bge-reranker-v2-m3` chấm từng cặp câu hỏi–đoạn văn và lấy tối đa 3 đoạn. Chưa lưu benchmark latency, nên không khẳng định đạt mốc 150 ms. |
| RAGAS và chẩn đoán lỗi | M4 — `evaluate_ragas()`, `failure_analysis()` | Tính faithfulness, answer relevancy, context precision và context recall; xếp Bottom-N theo điểm trung bình. Lần chạy mới cho Production: 0,6508 / 0,6786 / 0,9375 / 0,8167 theo thứ tự trên. `per_question` giúp kiểm tra cả điểm lẫn câu trả lời và contexts. |
| Contextual prepend, HyQA, metadata | M5 — `_enrich_single_call()`, `enrich_chunks()` | Một lời gọi `gpt-4o-mini` trả summary, câu hỏi giả định, câu bối cảnh và metadata. Pipeline hiện đưa câu bối cảnh vào `enriched_text`; summary và câu hỏi vẫn là các trường riêng của `EnrichedChunk`. Khi thiếu API key có fallback cục bộ. |

## Phần 2: Khó khăn & Cách giải quyết (Challenges & Debugging)

- **Lỗi kỹ thuật đã gặp:** Lần chạy trong sandbox báo `WinError 10013` khi tải `BAAI/bge-m3`. Sau đó `json.dump()` báo `TypeError: Object of type ndarray is not JSON serializable` khi lưu `per_question`. Đã đổi `contexts` sang danh sách chuỗi trước khi tạo `EvalResult`; lần chạy mới ghi đủ 20 câu cho cả hai báo cáo.
- **Lỗi quan sát từ kết quả:** Production thấp hơn baseline ở faithfulness (0,6508 so với 0,7667), answer relevancy (0,6786 so với 0,7691) và context recall (0,8167 so với 0,9000); context precision chỉ cao hơn 0,0042. Câu đổi mật khẩu và nghỉ phép thử việc bị trả “Không tìm thấy” dù tài liệu có đáp án. Một câu mua thiết bị 55 triệu trả Kế toán trưởng thay vì CEO vì lấy nhầm tài liệu tạm ứng.
- **Cách debug:** Đối chiếu `answer`, `contexts` và bốn metric trong `per_question` với tài liệu nguồn. Phát hiện chunk thử việc mất cụm “KHÔNG được”; top 3 câu mật khẩu chứa cả bản 90 và 120 ngày; hàng bảng phê duyệt trên 50 triệu không lọt top 3. Điểm RAGAS không thay thế đánh giá đúng/sai: câu 55 triệu sai nhưng faithfulness 1,0, còn câu Junior 17 triệu đúng nhưng faithfulness 0.
- **Kiến thức cần bổ sung:** Cách truy ngược từ child sang parent khi trả context, lọc phiên bản còn hiệu lực, xử lý bảng và phép tính nhiều bước, cùng benchmark latency theo từng tầng.

## Phần 3: Action Plan cho Project cá nhân (Application Plan)

### Project: Trợ lý tra cứu chính sách nội bộ

#### 1. Hiện trạng

- **Pipeline hiện tại:** Tài liệu Markdown/PDF có text layer → chunk cha–con → enrichment → BM25 + Qdrant Dense → RRF → Cross-Encoder top 3 → LLM → RAGAS.
- **Vấn đề:** Dữ liệu có chính sách cũ và mới cùng chủ đề; Production chưa vượt baseline ở ba chỉ số. Pipeline lưu `parent_id` nhưng chưa đưa parent vào ngữ cảnh trả lời. Bảng, câu phủ định và phép tính nhiều bước dễ bị thiếu; PDF scan vẫn cần OCR nếu muốn tra cứu.

#### 2. Kế hoạch cải tiến

1. **Chunking:** Giữ parent–child cho văn xuôi, dùng structure-aware cho bảng và điều khoản. Khi child được chọn, truy ngược parent để đưa câu hoàn chỉnh vào LLM; ghi `source`, phiên bản và ngày hiệu lực ở từng chunk.
2. **Retrieval:** Giữ BM25 + Dense + RRF; thêm lọc phiên bản hiện hành và theo dõi recall@20 trên 20 câu kiểm thử, đặc biệt các câu 2023/2024 và v1/v2.
3. **Reranking:** Giữ `BAAI/bge-reranker-v2-m3` trên top 20, so sánh top 3 với và không có rerank; đo p50/p95 latency bằng `benchmark_reranker()` trước khi đặt mục tiêu vận hành.
4. **Evaluation:** Dùng `per_question` đã lưu để so sánh với baseline trên cùng test set. Thêm kiểm tra thủ công cho đúng phiên bản, số liệu, phủ định và phép tính; xem lại các ca RAGAS chấm mâu thuẫn với đáp án chuẩn.
5. **Enrichment:** Thử contextual prepend trước; chạy A/B cho summary và HyQA trên chỉ mục tìm kiếm. Chỉ giữ kỹ thuật nào cải thiện recall/precision mà không làm tăng trả lời sai.

#### 3. Timeline triển khai

- **Tuần 1:** Truy ngược parent cho top child, gắn metadata phiên bản/ngày hiệu lực, giữ nguyên câu phủ định và hàng bảng trong context, bổ sung ca kiểm thử 55 triệu và 120 ngày.
- **Tuần 2:** Chạy A/B cho lọc phiên bản, reranking và enrichment; đo RAGAS, độ đúng thủ công và latency, rồi chọn cấu hình tốt hơn baseline.
