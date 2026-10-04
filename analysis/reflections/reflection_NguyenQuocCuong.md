# Individual Reflection — Lab 18: Production RAG

**Họ và tên:** Nguyễn Quốc Cường  
**Khóa:** K4 - Track 3B  
**Ngày hoàn thành:** 05/10/2026

---

## Phần 1: Mapping bài giảng (Lecture Mapping)

| Khái niệm | Module và hàm | Điều đã triển khai và quan sát |
|---|---|---|
| Semantic chunking | M1 — `chunk_semantic()` | Tách câu, mã hóa bằng `all-MiniLM-L6-v2`, ngắt khi cosine similarity dưới 0,85. Hàm đã viết nhưng pipeline chính không dùng chiến lược này; chưa có kết quả A/B riêng. |
| Parent–child và structure-aware chunking | M1 — `chunk_hierarchical()`, `chunk_structure_aware()` | Pipeline dùng child tối đa 256 ký tự nhưng chưa truy ngược parent trước khi trả lời. Câu “KHÔNG được nghỉ phép năm” bị cắt mất phủ định. Kiểm tra dữ liệu hiện tại cho thấy 26 parent cùng mang ID `parent_0` vì ID bắt đầu lại theo tài liệu; cần định danh duy nhất trước khi nối parent. Structure-aware cũng chưa được pipeline dùng cho bảng. |
| Hybrid retrieval và RRF | M2 — `BM25Search`, `DenseSearch`, `reciprocal_rank_fusion()` | BM25 và Dense (`BAAI/bge-m3`, Qdrant) được trộn bằng RRF. Baseline đã dùng cùng embedding và trả đúng nhiều câu nhờ giữ nguyên đoạn văn; chưa có ablation chứng minh hybrid tự nó cải thiện chất lượng trên bộ dữ liệu này. Chưa lọc chính sách hết hiệu lực. |
| Cross-encoder reranking | M3 — `CrossEncoderReranker.rerank()` | `BAAI/bge-reranker-v2-m3` xếp lại top 20 rồi chọn 3 child. Câu mật khẩu vẫn có bản cũ ở top 1 và bản mới ở top 2; rerank theo mức liên quan chưa giải quyết hiệu lực tài liệu. Chưa lưu số liệu latency. |
| RAGAS và chẩn đoán lỗi | M4 — `evaluate_ragas()`, `failure_analysis()` | Bản Production hiện tại đạt faithfulness 0,6875; answer relevancy 0,7592; context precision 0,9417; context recall 0,8500. Bottom-N xếp theo trung bình metric không đồng nghĩa Bottom-N câu trả lời sai: câu 55 triệu đúng CEO nhưng faithfulness 0; câu 20 ngày không lương trả đúng CEO nhưng thiếu chi tiết bảo hiểm chỉ có trong ground truth. |
| Contextual prepend, HyQA, metadata | M5 — `_enrich_single_call()`, `enrich_chunks()` | Một lời gọi `gpt-4o-mini` tạo summary, câu hỏi giả định, câu bối cảnh và metadata. Pipeline chỉ lập chỉ mục `enriched_text` gồm câu bối cảnh và child; summary và HyQA không đi vào tìm kiếm, metadata chưa dùng lọc phiên bản. Các câu bối cảnh khác nhau giữa hai snapshot Production, nên cần A/B cố định đầu ra để tách ảnh hưởng của enrichment. |

## Phần 2: Khó khăn & Cách giải quyết (Challenges & Debugging)

- **Lỗi kỹ thuật đã gặp:** Lần chạy trước trong sandbox báo `WinError 10013` khi tải `BAAI/bge-m3`. Sau đó `json.dump()` báo `TypeError: Object of type ndarray is not JSON serializable` khi lưu `per_question`. Đã đổi `contexts` sang danh sách chuỗi trước khi tạo `EvalResult`; hai báo cáo hiện lưu đủ 20 câu. Bản Production đã chạy lại sau bản baseline, nên cần lưu phiên bản cấu hình và thời điểm chạy trong các lần so sánh tiếp theo.
- **Kết quả mới:** Production thấp hơn baseline ở faithfulness (0,6875 so với 0,7667), answer relevancy (0,7592 so với 0,7691) và context recall (0,8500 so với 0,9000); context precision cao hơn 0,0083. Câu đổi mật khẩu và phép năm thử việc vẫn trả “Không tìm thấy”. Câu PVI của nhân viên thử việc nay trả đúng, và câu 55 triệu nay trả đúng CEO. Câu tạm ứng vẫn trả phí cả tháng 300.000 VNĐ trong khi ground truth giả sử quy đổi 5 ngày.
- **Cách debug:** Đối chiếu đáp án và ba context của từng câu với tài liệu nguồn, rồi mới đọc metric. Chunk thử việc mất “KHÔNG được”; câu mật khẩu có cả 90 và 120 ngày nhưng thiếu nhãn hiệu lực ngay ở đoạn mang con số. Ngược lại, context câu 55 triệu hiện có nguyên hàng bảng CEO, song RAGAS cho faithfulness 0. Câu 20 ngày không lương trả đúng CEO nhưng bị trừ vì ground truth còn chứa thông tin bảo hiểm không được hỏi. Câu Junior 17 triệu cũng đúng theo phép nhân 20 triệu × 85% dù faithfulness 0.
- **Điều cần học tiếp:** Nối child với parent bằng ID duy nhất, giữ nguyên câu và hàng bảng, xác định hiệu lực phiên bản, tách câu hỏi nhiều ý, kiểm chứng phép tính và xây thước đo đúng/sai riêng bên cạnh RAGAS. Cần đo latency từng tầng thay vì suy từ độ phức tạp module.

## Phần 3: Action Plan cho Project cá nhân (Application Plan)

### Project: Trợ lý tra cứu chính sách nội bộ

#### 1. Hiện trạng

- **Pipeline hiện tại:** Tài liệu Markdown/PDF có text layer → child của chunk cha–con → contextual prepend → BM25 + Qdrant Dense → RRF → Cross-Encoder top 3 → LLM → RAGAS. Summary và HyQA được tạo nhưng chưa tham gia chỉ mục.
- **Vấn đề:** Dữ liệu có chính sách cũ và mới cùng chủ đề; Production chưa vượt baseline ở ba chỉ số. `parent_id` hiện trùng giữa các tài liệu và parent chưa được đưa vào ngữ cảnh trả lời. Bảng, câu phủ định và các điều kiện tính phí dễ bị tách; hai PDF scan đang bị bỏ qua vì chưa có OCR. Điểm RAGAS có những ca mâu thuẫn với đáp án đúng/sai thủ công.

#### 2. Kế hoạch cải tiến

1. **Chunking:** Tạo ID parent duy nhất theo `source` và vị trí; lưu bảng tra parent. Không cắt giữa câu phủ định hoặc hàng bảng; dùng structure-aware cho các mục chính sách và bảng ngưỡng. Khi chọn child, thử trả parent/section và đo lại độ đúng.
2. **Retrieval:** So sánh dense-only baseline với hybrid trên cùng index và cùng chunk trước khi kết luận lợi ích của BM25/RRF. Trích phiên bản, ngày hiệu lực, trạng thái thay thế từ header; lọc bản cũ cho câu hỏi về chính sách hiện hành. Theo dõi recall@20 và nguồn đúng cho câu 120 ngày, 55 triệu và Junior.
3. **Reranking:** So sánh top 3 trước/sau `BAAI/bge-reranker-v2-m3`, kiểm tra riêng trường hợp bản cũ đứng trên bản mới. Đo thời gian tìm kiếm, rerank, sinh đáp án; `benchmark_reranker()` hiện chỉ trả avg/min/max, muốn p50/p95 phải lưu từng lần đo rồi tự tính.
4. **Evaluation:** Lưu cấu hình, đầu ra enrichment, câu trả lời và context của từng lượt chạy. Chấm đúng/sai thủ công theo từng ý bắt buộc bên cạnh bốn metric RAGAS; tách câu CEO phê duyệt khỏi phần bảo hiểm, làm rõ chính sách pro-rata trước khi dùng câu tạm ứng làm ground truth. Lặp lại đánh giá để thấy độ dao động.
5. **Enrichment:** Chạy ablation không enrichment / contextual prepend / summary / HyQA trên cùng tập chunk và cùng câu hỏi. Chỉ đưa summary/HyQA vào index khi có thiết kế ánh xạ về văn bản gốc, rồi giữ phương án cải thiện độ đúng và retrieval.

#### 3. Timeline triển khai

- **Tuần 1:** Sửa ID parent, truy ngược parent cho top child, giữ nguyên câu phủ định và hàng bảng, thêm metadata hiệu lực. Chuẩn hóa ground truth câu tạm ứng và tách câu nghỉ không lương thành hai ý kiểm tra riêng.
- **Tuần 2:** Chạy ablation từng thay đổi trên cùng 20 câu và một nhóm câu mới chưa dùng để tối ưu; lưu nhiều lượt chạy, so sánh độ đúng từng ý, RAGAS và latency với dense-only baseline trước khi chọn cấu hình.
