# Failure Analysis — Lab 18: Production RAG

**Họ và tên học viên:** Nguyễn Quốc Cường
**Khóa:** K4 - Track 3B

---

## RAGAS Scores

| Metric | Naive Baseline | Production | Δ |
|--------|---------------|------------|---|
| Faithfulness | 0,7667 | 0,6508 | −0,1158 |
| Answer Relevancy | 0,7691 | 0,6786 | −0,0905 |
| Context Precision | 0,9333 | 0,9375 | +0,0042 |
| Context Recall | 0,9000 | 0,8167 | −0,0833 |

Nguồn: `reports/naive_baseline_report.json` và `reports/ragas_report.json` của lần chạy mới; cả hai có `per_question` cho 20 câu. Production chỉ nhỉnh hơn 0,0042 ở context precision. Faithfulness và answer relevancy đều dưới mục tiêu 0,70; chưa có bằng chứng cải thiện tổng thể so với baseline. Đặc biệt, có ca trả lời sai nhưng faithfulness vẫn đạt 1,0, và ca trả lời đúng phép tính nhưng faithfulness bằng 0; vì vậy cần đối chiếu thủ công với đáp án chuẩn cùng ngữ cảnh.

Trong `src/pipeline.py`, hệ thống lập chỉ mục child và gửi `child.text` cho LLM, nhưng chưa dùng `parent_id` để lấy lại văn bản parent. Điều này giải thích vì sao lời phủ định, đầu mục hoặc một hàng bảng có thể nằm ngoài ba đoạn được chọn dù parent chứa đầy đủ thông tin.

## Bottom-5 Failures

### #1
- **Question:** Bao lâu phải đổi mật khẩu một lần?
- **Expected:** 120 ngày theo `mat_khau_v2.md` hiện hành; 90 ngày trong `mat_khau_v1.md` đã bị thay thế.
- **Got / câu trả lời có đúng không?** “Không tìm thấy.” — sai vì tài liệu có đáp án. Điểm trung bình 0,3958; `faithfulness` và `answer_relevancy` đều bằng 0.
- **Contexts có đáp án không?** Có. Top 1 chứa **90 ngày** của bản cũ; top 2 chứa **120 ngày** của bản mới; top 3 nói bản cũ đã bị thay thế. Phần đầu chunk không giữ đầy đủ nhãn phiên bản, khiến hai mốc dễ bị trộn.
- **Query rewrite?** Có ích: “Theo chính sách mật khẩu v2.0 hiệu lực 01/07/2024, chu kỳ đổi mật khẩu là bao nhiêu ngày?”
- **Error Tree:** Output sai → Context có đáp án nhưng xung đột → Query thiếu phiên bản → cần giải quyết hiệu lực trước khi sinh câu trả lời.
- **Root cause:** Bản cũ đứng trước bản mới trong top 3; LLM chọn từ chối dù bản mới và thông báo thay thế đều đã được truy xuất. Baseline trả đúng 120 ngày.
- **Suggested fix / module:** M1 giữ nhãn phiên bản trong mỗi child; M2 lọc bản đã hết hiệu lực; M3 ưu tiên bản hiện hành; `src/pipeline.py` yêu cầu chọn nguồn còn hiệu lực và nêu 120 ngày.

### #2
- **Question:** Nhân viên thử việc có được nghỉ phép năm không?
- **Expected:** Không; nếu cần nghỉ riêng thì xin nghỉ không lương và được trưởng phòng phê duyệt, theo `thu_viec.md`.
- **Got / câu trả lời có đúng không?** “Không tìm thấy.” — sai. Điểm trung bình 0,5000; `faithfulness` và `answer_relevancy` đều bằng 0.
- **Contexts có đáp án không?** Chỉ một phần. Top 1 bắt đầu bằng “nghỉ phép năm**. Trường hợp cần nghỉ...”, nhưng thiếu cụm phủ định “KHÔNG được” nằm ở đầu câu gốc. Hai đoạn còn lại là chính sách phép năm 2023 cho nhân viên chính thức, không trả lời về thử việc.
- **Query rewrite?** Không cần; câu hỏi nêu rõ đối tượng và quyền lợi.
- **Error Tree:** Output sai → Context bị cụt mất từ phủ định → Query rõ → sửa cắt đoạn và truy xuất.
- **Root cause:** M1 chia giữa câu quan trọng; M2/M3 chỉ đưa nửa sau vào top 3. Baseline giữ câu đầy đủ và trả lời đúng.
- **Suggested fix / module:** M1 không cắt giữa câu, ưu tiên đoạn structure-aware cho mục quyền lợi; M2/M3 đưa chunk chứa “KHÔNG được nghỉ phép năm” vào top 3.

### #3
- **Question:** Nhân viên thử việc có được hưởng bảo hiểm sức khỏe PVI không?
- **Expected:** Không; chỉ tham gia bảo hiểm xã hội bắt buộc, chưa được hưởng PVI, theo `thu_viec.md`.
- **Got / câu trả lời có đúng không?** “Không tìm thấy.” — sai. Điểm trung bình 0,5000; `faithfulness` và `answer_relevancy` đều bằng 0.
- **Contexts có đáp án không?** Có. Top 1 ghi rõ nhân viên thử việc “chưa được hưởng gói bảo hiểm sức khỏe PVI” và chỉ tham gia bảo hiểm xã hội bắt buộc. Top 2 cũng nói PVI dành cho nhân viên chính thức.
- **Query rewrite?** Không cần; trạng thái “thử việc” và gói PVI đã rõ.
- **Error Tree:** Output sai → Context có đáp án trực tiếp → Query rõ → lỗi ở bước sinh câu trả lời hoặc xử lý phủ định.
- **Root cause:** LLM từ chối trả lời dù bằng chứng đứng ở top 1; không thể quy lỗi cho retrieval trong ca này.
- **Suggested fix / module:** Trong `src/pipeline.py`, yêu cầu trả lời “Có/Không” từ câu chứa phủ định rồi giải thích ngắn; thêm kiểm thử cho câu hỏi về ngoại lệ. Baseline cũng trả “Không tìm thấy”, nên đây là lỗi chung.

### #4
- **Question:** Nhân viên tạm ứng 15 triệu, sau 20 ngày mới thanh toán. Bị phạt bao nhiêu?
- **Expected:** Theo `test_set.json`, quá hạn 5 ngày; phí 2%/tháng trên 15 triệu là 300.000 VNĐ/tháng, quy đổi pro-rata 5 ngày khoảng 50.000 VNĐ. `tam_ung.md` chỉ ghi 2%/tháng, chưa nói rõ quy tắc pro-rata.
- **Got / câu trả lời có đúng không?** Trả 300.000 VNĐ là số phí **cả tháng**, không phải số phí cho 5 ngày quá hạn theo đáp án kiểm thử. Điểm trung bình 0,6179; `faithfulness` thấp nhất (0,1667).
- **Contexts có đáp án không?** Có thời hạn 15 ngày và mức phí 2%/tháng, nhưng không có hướng dẫn rõ về cách chia phí theo ngày. Thông tin nằm ở hai chunk liên tiếp.
- **Query rewrite?** Câu gốc rõ; có thể nêu rõ “phí cho 5 ngày quá hạn, giả sử 30 ngày/tháng” để phép tính xác định.
- **Error Tree:** Output sai số tiền cần trả → Context có tỷ lệ nhưng thiếu quy tắc pro-rata → Query có thể làm rõ giả định → sửa dữ liệu và phép tính.
- **Root cause:** LLM áp phí cả tháng cho 5 ngày; đáp án chuẩn cũng cần một giả định pro-rata chưa được viết trong chính sách nguồn.
- **Suggested fix / module:** Bổ sung quy tắc pro-rata vào `tam_ung.md` và test set; trong `src/pipeline.py`, tách thời gian quá hạn rồi tính `15.000.000 × 2% × 5/30 ≈ 50.000`, nêu rõ giả định. M1 nên giữ thời hạn và phạt trong cùng parent context.

### #5
- **Question:** Khi phát hiện malware trên máy, nhân viên có nên tự xử lý không?
- **Expected:** Tuyệt đối không tự xử lý; báo cáo trong 1 giờ qua helpdesk@cty.vn hoặc hotline CNTT; tự xử lý là vi phạm nghiêm trọng.
- **Got / câu trả lời có đúng không?** “Không nên tự xử lý ... chờ hướng dẫn từ đội CNTT.” Đúng ý có/không nhưng thiếu nghĩa vụ báo cáo và thời hạn; điểm trung bình 0,6667. RAGAS chấm `answer_relevancy=0` dù câu trả lời bám câu hỏi, cần kiểm tra lại metric này bằng đánh giá thủ công.
- **Contexts có đáp án không?** Có phần “không tự ý xử lý” và hậu quả vi phạm. Top 1 bắt đầu giữa đoạn sau thông tin báo cáo trong 1 giờ, nên bằng chứng cho thời hạn/email bị mất; context recall là 0,6667.
- **Query rewrite?** Câu gốc rõ cho phần có/không; nếu muốn câu trả lời đầy đủ, thêm “phải báo ai và trong bao lâu?”
- **Error Tree:** Output đúng một phần → Context thiếu đầu đoạn về báo cáo → Query hỏi ngắn → kết hợp context liền kề khi trả lời.
- **Root cause:** M1/M2/M3 không giữ phần đầu của quy định cùng top 3. Điểm answer relevancy bằng 0 không phản ánh đầy đủ chất lượng câu trả lời này.
- **Suggested fix / module:** M1 giữ toàn bộ mục “Báo cáo sự cố” trong parent; M2/M3 trả parent hoặc đoạn kề; `src/pipeline.py` nêu cả việc không tự xử lý và kênh/thời hạn báo cáo.

## Case Study (cho presentation)

**Question chọn phân tích:** Bao lâu phải đổi mật khẩu một lần? — xung đột bản cũ v1.0 (90 ngày) và bản hiện hành v2.0 (120 ngày).

**Error Tree walkthrough:**
1. Output đúng? → Không: mô hình trả “Không tìm thấy”, trong khi đáp án là 120 ngày.
2. Context đúng? → Có đáp án, nhưng top 1 là 90 ngày của bản cũ, top 2 là 120 ngày của bản mới; thông báo thay thế ở top 3.
3. Query rewrite OK? → Câu gốc thiếu phiên bản; thêm “theo chính sách hiện hành v2.0” để loại nhập nhằng.
4. Fix ở bước: M1 giữ metadata phiên bản theo child; M2 loại bản hết hiệu lực; M3 xếp bản hiện hành trước; `src/pipeline.py` chọn nguồn hiệu lực và nêu 120 ngày.

**Nếu có thêm 1 giờ, sẽ optimize:**
- Ưu tiên hai thử nghiệm: giữ nguyên câu phủ định khi chunk và lọc phiên bản cũ trước retrieval. Sau đó chạy lại cùng 20 câu, kiểm tra riêng #1, #2, #3 và ca mua thiết bị 55 triệu.

## Ca nghiêm trọng ngoài Bottom-5 và giới hạn của RAGAS

- **Mua thiết bị 55 triệu:** Production trả “Cần thêm phê duyệt Kế toán trưởng”, nhưng đáp án đúng là **CEO**. Top 3 không chứa hàng bảng “Trên 50.000.000 VNĐ | CEO”; lại chứa quy định Kế toán trưởng của tài liệu *tạm ứng*. RAGAS vẫn cho `faithfulness=1,0` và `context_precision≈1,0`, trong khi `context_recall=0`. Đây là lỗi retrieval và trộn nghiệp vụ; cần giữ nguyên bảng ở M1, lọc nguồn mua sắm ở M2 và kiểm tra đúng hàng ngưỡng ở M3.
- **Lương thử việc Junior:** Production trả đúng **17 triệu VNĐ**, hai ngữ cảnh có trần Junior 20 triệu và tỷ lệ thử việc 85%, nhưng RAGAS cho `faithfulness=0`. Đây là dấu hiệu metric có thể đánh giá sai phép tính ghép hai đoạn; cần thêm kiểm tra thủ công hoặc metric số học.
- **Senior 9 năm:** Câu trả lời nêu 18 ngày nhưng giải thích sai thành “12 + 6” và bỏ khung lương 20–35 triệu. Top 3 chỉ có tài liệu phép năm, gồm cả bản 2023; không có bảng lương. Cần truy xuất riêng hai vế và loại chính sách cũ.
