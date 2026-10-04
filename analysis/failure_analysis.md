# Failure Analysis — Lab 18: Production RAG

**Họ và tên học viên:** Nguyễn Quốc Cường
**Khóa:** K4 - Track 3B

---

## RAGAS Scores

| Metric | Naive Baseline | Production | Δ |
|--------|---------------|------------|---|
| Faithfulness | 0,7667 | 0,6875 | −0,0792 |
| Answer Relevancy | 0,7691 | 0,7592 | −0,0099 |
| Context Precision | 0,9333 | 0,9417 | +0,0083 |
| Context Recall | 0,9000 | 0,8500 | −0,0500 |

Nguồn: `reports/naive_baseline_report.json` và bản `reports/ragas_report.json` hiện tại, mỗi bản có 20 câu trong `per_question`. Báo cáo Production đã được chạy lại, còn báo cáo baseline là bản đã lưu trước đó; do đó đây là so sánh hai snapshot trên cùng test set, không phải thí nghiệm lặp nhiều lần. Production thấp hơn baseline ở 3/4 metric, nhưng chênh lệch answer relevancy chỉ khoảng 0,01. Không suy ra tỷ lệ trả lời đúng từ trung bình RAGAS: câu 55 triệu trả **đúng CEO** nhưng faithfulness bằng 0; câu nghỉ không lương 20 ngày trả **đúng người phê duyệt** nhưng bị trừ điểm vì thiếu chi tiết bảo hiểm trong ground truth.

Baseline dùng cùng mô hình embedding và LLM, trên 26 tài liệu ngắn tạo 57 đoạn theo paragraph. Production tạo 100 child, mỗi child tối đa 256 ký tự; `src/pipeline.py` lập chỉ mục và gửi child đã enrich cho LLM nhưng chưa truy ngược parent. Trong kho tài liệu này, cả 26 parent đều mang ID `parent_0` vì ID bắt đầu lại ở từng tài liệu. Đây là lỗi định danh cần sửa trước khi thêm parent lookup. Summary và câu hỏi giả định của M5 cũng chưa đi vào chỉ mục; chỉ câu bối cảnh được prepend vào child.

## Bottom-5 theo RAGAS

Thứ hạng dưới đây dùng trung bình bốn metric RAGAS. Nhãn “failure” là thứ hạng điểm, cần kiểm tra đáp án thủ công trước khi quy lỗi cho hệ thống.

### #1 — Đổi mật khẩu (0,3958; trả sai)

- **Expected / got:** 120 ngày theo `mat_khau_v2.md` hiện hành; Production trả “Không tìm thấy.” Baseline trả đúng 120 ngày.
- **Bằng chứng:** Top 1 có 90 ngày bản cũ, top 2 có 120 ngày bản mới, top 3 nói bản cũ đã bị thay thế. Hai chunk chứa con số không mang nhãn phiên bản ở chính đoạn được gửi cho LLM; `context_recall=1`, nên đây không đơn thuần là thiếu kết quả truy xuất.
- **Error Tree:** Output sai → context chứa đáp án nhưng xung đột phiên bản → câu hỏi không nêu phiên bản → phải xác định nguồn còn hiệu lực trước khi trả lời.
- **Fix:** Gắn `source`, phiên bản, trạng thái hiệu lực cho từng child; lọc hoặc ưu tiên v2.0 trước rerank. Truy lại parent hoặc header để LLM nhìn thấy quy định 120 ngày cùng nhãn hiện hành. Có thể viết lại câu hỏi để làm rõ phiên bản, nhưng không nên bắt người dùng luôn phải nêu v2.0.

### #2 — Phép năm của nhân viên thử việc (0,4583; trả sai)

- **Expected / got:** Nhân viên thử việc **không** được nghỉ phép năm; Production trả “Không tìm thấy.” Baseline trả đúng.
- **Bằng chứng:** Top 1 bắt đầu bằng “nghỉ phép năm**”, mất cụm “Nhân viên thử việc **KHÔNG được” ở đầu câu gốc. Hai context còn lại nói về phép năm của nhân viên chính thức theo bản 2024; `context_recall=1` của RAGAS không phát hiện mất phủ định ở đoạn quan trọng.
- **Error Tree:** Output sai → context top 1 thiếu mệnh đề quyết định → câu hỏi rõ → lỗi cắt child và không mở rộng ngữ cảnh.
- **Fix:** Không cắt giữa câu hoặc hàng bảng; sau khi tìm child, gửi parent/section chứa nguyên câu phủ định vào LLM. Thêm kiểm tra tự động cho câu hỏi Có/Không có phủ định.

### #3 — Phí tạm ứng quá hạn (0,6584; đáp án thiếu cơ sở tính)

- **Expected / got:** Test set giả sử tính phí pro-rata 5 ngày quá hạn, khoảng 50.000 VNĐ; Production trả 300.000 VNĐ, tức 2% cho cả tháng. Baseline cũng trả 300.000 VNĐ.
- **Bằng chứng:** Top 3 có hạn thanh toán 15 ngày và phí 2%/tháng, nhưng hai quy định nằm ở các child khác nhau. `tam_ung.md` không nói phí tháng lẻ tính pro-rata, làm tròn hay thu trọn tháng.
- **Error Tree:** Con số khác ground truth → có thời hạn và tỷ lệ nhưng thiếu quy tắc quy đổi → chưa thể chốt duy nhất mức phí 5 ngày từ tài liệu nguồn.
- **Fix:** Trước tiên làm rõ chính sách và ground truth về tháng lẻ. Nếu chính sách xác nhận pro-rata theo 30 ngày, đáp án là `15.000.000 × 2% × 5/30 = 50.000 VNĐ`; khi đó mới kiểm tra logic tính toán của pipeline. Không tự thêm quy tắc này vào nguồn như thể đã được ban hành.

### #4 — Nghỉ không lương 20 ngày (0,6959; trả đúng ý hỏi)

- **Expected / got:** Cần CEO phê duyệt; Production trả đúng CEO. Ground truth còn thêm nghĩa vụ tự đóng phần bảo hiểm khi nghỉ trên 14 ngày, trong khi câu hỏi chỉ hỏi người phê duyệt.
- **Bằng chứng:** Top 1 có nguyên quy định “16-30 ngày → CEO”; phần bảo hiểm ở đoạn sau không nằm trọn trong top 3. RAGAS cho `faithfulness=0,5`, `context_recall=0,5` dù câu trả lời chính đúng.
- **Error Tree:** Output đúng câu hỏi → ngữ cảnh đủ cho người phê duyệt → metric phạt phần thông tin bổ sung trong ground truth.
- **Fix:** Tách bài kiểm tra “ai phê duyệt?” và “nghỉ 20 ngày ảnh hưởng bảo hiểm thế nào?”, hoặc yêu cầu câu hỏi hỏi cả hai. Không xếp ca này là lỗi trả lời sai.

### #5 — Mua thiết bị 55 triệu (0,7026; trả đúng nhưng metric bất thường)

- **Expected / got:** Trên 50 triệu cần CEO; Production và baseline đều trả đúng CEO.
- **Bằng chứng:** Top 1 hiện có nguyên hàng bảng “Trên 50.000.000 VNĐ | Tổng Giám đốc (CEO)”. Dù vậy RAGAS cho `faithfulness=0`, còn `context_precision≈1` và `context_recall=1`. Top 3 vẫn lẫn quy định Kế toán trưởng của tài liệu tạm ứng, nhưng lần này LLM không dùng nhầm.
- **Error Tree:** Output đúng và context hỗ trợ trực tiếp → điểm faithfulness mâu thuẫn với bằng chứng → cần kiểm tra lần chấm, không quy lỗi cho truy xuất ở ca này.
- **Fix:** Ghi nhận đúng/sai thủ công cạnh RAGAS; kiểm tra lại evaluator trên cặp answer/context này. Tiếp tục lọc nhầm tài liệu tạm ứng để giảm rủi ro ở những lần chạy sau.

## Case Study (cho presentation)

**Question chọn phân tích:** Bao lâu phải đổi mật khẩu một lần? — xung đột bản cũ v1.0 (90 ngày) và bản hiện hành v2.0 (120 ngày).

**Error Tree walkthrough:**
1. Output đúng? → Không: mô hình trả “Không tìm thấy”, trong khi đáp án là 120 ngày.
2. Context đúng? → Có đáp án, nhưng top 1 là 90 ngày của bản cũ, top 2 là 120 ngày của bản mới; thông báo thay thế ở top 3.
3. Query rewrite OK? → Câu gốc thiếu phiên bản; thêm “theo chính sách hiện hành v2.0” để loại nhập nhằng.
4. Fix ở bước: M1 giữ metadata phiên bản và parent có ID duy nhất; M2 loại bản hết hiệu lực; M3 xếp bản hiện hành trước; `src/pipeline.py` gửi đoạn có nhãn hiệu lực và trả 120 ngày.

**Nếu có thêm 1 giờ, sẽ thử nghiệm:**
- Giữ nguyên câu phủ định và hàng bảng khi chunk; lọc phiên bản cũ trước retrieval. Chạy lại cùng 20 câu và ghi cả điểm RAGAS lẫn đáp án đúng/sai cho câu mật khẩu, phép năm thử việc, PVI và 55 triệu. Chạy nhiều lượt hoặc lưu đầu ra cố định để tách thay đổi hệ thống khỏi dao động của LLM/evaluator.

## Các ca ngoài Bottom-5 và giới hạn của RAGAS

- **PVI của nhân viên thử việc:** Production hiện trả đúng “chưa được hưởng PVI”, `faithfulness=1` và `answer_relevancy≈0,992`; baseline vẫn trả “Không tìm thấy”. Đây là cải thiện rõ trong snapshot mới, không còn thuộc Bottom-5.
- **Lương thử việc Junior:** Production trả đúng **17 triệu VNĐ**. Top 3 có trần Junior 20 triệu và tỷ lệ 85%, nhưng RAGAS cho `faithfulness=0`. Cần kiểm tra thủ công phép tính `20.000.000 × 85%` và không dùng riêng faithfulness để kết luận đáp án sai.
- **Senior 9 năm:** Production nay giải thích đúng **15 + 3 = 18 ngày**, nhưng chưa đưa ra khung lương **20–35 triệu** vì top 3 chỉ có tài liệu phép năm, gồm cả bản cũ. Đây là câu trả lời thiếu một vế; cần truy xuất riêng hai phần của câu hỏi rồi tổng hợp.
- **Malware:** Production trả đúng “không tự xử lý”; `answer_relevancy≈0,853`, không còn bằng 0. Câu trả lời vẫn thiếu nghĩa vụ báo cáo trong 1 giờ vì đoạn đầu quy định không có trong top 3.
- **Laptop 30 triệu:** Production trả đúng Director và xác nhận cấu hình từ CNTT, nhưng bỏ yêu cầu **ít nhất 3 báo giá** có trong ground truth. Phần này có trong tài liệu nguồn nhưng bị tách khỏi context đầy đủ; nên kiểm tra câu trả lời nhiều điều kiện theo từng ý.
