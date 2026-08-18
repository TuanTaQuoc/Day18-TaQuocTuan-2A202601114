# Failure Analysis - Lab 18: Production RAG

**Nhóm:** Cá nhân  
**Thành viên:** Tạ Quốc Tuấn - MSSV: 2A202601114

---

## RAGAS Scores

| Metric | Naive Baseline | Production | Δ |
|--------|---------------|------------|---|
| Faithfulness | 0.8083 | 0.5975 | -0.2108 |
| Answer Relevancy | 0.7108 | 0.6312 | -0.0797 |
| Context Precision | 0.9250 | 0.9333 | +0.0083 |
| Context Recall | 0.9250 | 0.7417 | -0.1833 |

## Bottom-5 Failures

### #1
- **Question:** Lương thử việc của nhân viên Junior mức cao nhất là bao nhiêu?
- **Expected:** Junior cao nhất là 20.000.000 VNĐ/tháng; lương thử việc = 85% x 20.000.000 = 17.000.000 VNĐ/tháng.
- **Got:** Không tìm thấy.
- **Worst metric:** Faithfulness = 0.0
- **Error Tree:** Output sai -> Context đúng? Không đủ context bảng lương/thu việc -> Query OK? Có -> Retrieval thiếu chunk liên quan đến `bang_luong_2024.md` hoặc `thu_viec.md`.
- **Root cause:** Pipeline abstain vì context top-k sau rerank không chứa đủ cả mức lương Junior và công thức 85%.
- **Suggested fix:** Tăng recall bằng lấy nhiều candidate hơn trước rerank, thêm metadata filter cho nhóm tài liệu finance/hr, và ghép parent context khi child chứa thông tin lương.

### #2
- **Question:** Thông tin lương thuộc cấp độ phân loại dữ liệu nào?
- **Expected:** Thông tin lương là dữ liệu Bí mật/cấp 3, cần mã hóa khi truyền và hạn chế truy cập theo need-to-know.
- **Got:** Không tìm thấy.
- **Worst metric:** Faithfulness = 0.0
- **Error Tree:** Output sai -> Context đúng? Thiếu liên kết giữa tài liệu lương và phân loại dữ liệu -> Query OK? Có -> Retrieval chưa nối được hai chính sách.
- **Root cause:** Câu hỏi cần tổng hợp chéo `bang_luong_2024.md` và `phan_loai_du_lieu.md`; top context không bao phủ đủ hai nguồn.
- **Suggested fix:** Thêm query expansion/HyQA cho các câu hỏi cross-document và tăng `RERANK_TOP_K` để giữ nhiều context sau rerank.

### #3
- **Question:** Một nhân viên Senior có 9 năm thâm niên được nghỉ bao nhiêu ngày phép năm và lương trong khoảng nào?
- **Expected:** 18 ngày phép; lương Senior P3-P4 là 20-35 triệu VNĐ/tháng.
- **Got:** Không tìm thấy.
- **Worst metric:** Faithfulness = 0.0
- **Error Tree:** Output sai -> Context đúng? Không đủ cả chính sách phép năm và bảng lương -> Query OK? Có nhưng multi-hop -> Retrieval thiếu một trong hai mảnh thông tin.
- **Root cause:** Câu hỏi yêu cầu tính toán và tổng hợp hai tài liệu, trong khi pipeline ưu tiên precision nên recall bị giảm.
- **Suggested fix:** Dùng decomposition cho câu hỏi nhiều ý: truy vấn riêng "Senior lương" và "9 năm thâm niên phép năm", rồi merge context trước khi answer.

### #4
- **Question:** Thâm niên bao nhiêu năm thì được cộng thêm ngày phép?
- **Expected:** Chính sách v2024: từ 3 năm trở lên, cộng 1 ngày cho mỗi 3 năm; v2023 là 5 năm nhưng đã thay thế.
- **Got:** Không tìm thấy.
- **Worst metric:** Faithfulness = 0.0
- **Error Tree:** Output sai -> Context đúng? Thiếu hoặc không ưu tiên version v2024 -> Query OK? Có -> Rerank chưa xử lý tốt tài liệu có phiên bản cũ/mới.
- **Root cause:** Có nhiều policy nghỉ phép theo version; pipeline cần ưu tiên tài liệu hiện hành.
- **Suggested fix:** Extract metadata `version`/`effective_date`, boost tài liệu v2024, và thêm rule loại bỏ tài liệu bị thay thế khi trả lời.

### #5
- **Question:** Nhân viên được nghỉ bao nhiêu ngày phép năm?
- **Expected:** Theo chính sách hiện hành v2024, nhân viên được nghỉ 15 ngày phép năm có lương; v2023 là 12 ngày nhưng đã bị thay thế.
- **Got:** Nhân viên được hưởng 12 ngày phép năm có lương.
- **Worst metric:** Context Recall = 0.0
- **Error Tree:** Output sai -> Context đúng? Không, context nghiêng về chính sách cũ -> Query OK? Có -> Version conflict không được xử lý.
- **Root cause:** Retrieval/rerank chọn chunk v2023 có lexical overlap cao hơn chunk v2024.
- **Suggested fix:** Thêm metadata về năm hiệu lực, rerank có feature freshness, và prompt bắt buộc kiểm tra tài liệu mới nhất khi có nhiều phiên bản.

## Case Study (cho presentation)

**Question chọn phân tích:** Nhân viên được nghỉ bao nhiêu ngày phép năm?

**Error Tree walkthrough:**
1. Output đúng? -> Không. Output trả lời 12 ngày, trong khi chính sách hiện hành v2024 là 15 ngày.
2. Context đúng? -> Không đủ. Context recall = 0.0 cho thấy chunk chứa policy v2024 không được đưa vào answer context.
3. Query rewrite OK? -> Query ngắn và rõ, nhưng thiếu dấu hiệu "hiện hành/mới nhất", nên retrieval dễ bắt nhầm version cũ.
4. Fix ở bước: Metadata enrichment + rerank. Cần đánh dấu `version=2024`, `status=current`, và boost tài liệu hiện hành.

**Nếu có thêm 1 giờ, sẽ optimize:**
- Thêm metadata version/effective_date cho các policy có nhiều phiên bản.
- Tăng số context sau rerank từ 3 lên 5 cho câu hỏi có dấu hiệu multi-hop hoặc version conflict.
- Viết prompt answer yêu cầu nêu rõ nếu có chính sách cũ bị thay thế.
