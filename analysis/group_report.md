# Group Report - Lab 18: Production RAG

**Nhóm:** Cá nhân  
**Ngày:** 18/08/2026  
**Sinh viên:** Tạ Quốc Tuấn - MSSV: 2A202601114

## Thành viên & Phân công

| Tên | Module | Hoàn thành | Tests pass |
|-----|--------|-----------|-----------|
| Tạ Quốc Tuấn | M1: Chunking | ☑ | 13/13 |
| Tạ Quốc Tuấn | M2: Hybrid Search | ☑ | 5/5 |
| Tạ Quốc Tuấn | M3: Reranking | ☑ | 5/5 |
| Tạ Quốc Tuấn | M4: Evaluation | ☑ | 4/4 |
| Tạ Quốc Tuấn | M5: Enrichment | ☑ | 10/10 |

Tổng test: 37/37 passed.

## Kết quả RAGAS

| Metric | Naive | Production | Δ |
|--------|-------|-----------|---|
| Faithfulness | 0.8083 | 0.5975 | -0.2108 |
| Answer Relevancy | 0.7108 | 0.6312 | -0.0797 |
| Context Precision | 0.9250 | 0.9333 | +0.0083 |
| Context Recall | 0.9250 | 0.7417 | -0.1833 |

## Key Findings

1. **Biggest improvement:** Context Precision tăng nhẹ từ 0.9250 lên 0.9333 nhờ hybrid search + reranking lọc bớt context nhiễu.
2. **Biggest challenge:** Faithfulness và Context Recall giảm do pipeline production đôi lúc trả "Không tìm thấy" hoặc chọn nhầm policy cũ khi tài liệu có nhiều phiên bản.
3. **Surprise finding:** Pipeline phức tạp hơn không tự động tốt hơn baseline nếu metadata version, parent retrieval và prompt xử lý conflict chưa đủ mạnh.

## Presentation Notes (5 phút)

1. RAGAS scores (naive vs production): Production đạt Context Precision 0.9333 nhưng Faithfulness 0.5975 và Context Recall 0.7417, thấp hơn baseline ở 3/4 metrics.
2. Biggest win - module nào, tại sao: M2 + M3 cải thiện precision nhờ BM25/Dense/RRF và CrossEncoder rerank, giúp context top-k ít nhiễu hơn.
3. Case study - 1 failure, Error Tree walkthrough: Câu "Nhân viên được nghỉ bao nhiêu ngày phép năm?" trả 12 ngày do chọn nhầm policy v2023; fix bằng metadata version và boost policy hiện hành v2024.
4. Next optimization nếu có thêm 1 giờ: thêm `version/status/effective_date` metadata, tăng recall cho câu hỏi multi-hop, và cập nhật prompt bắt buộc kiểm tra tài liệu mới nhất.
