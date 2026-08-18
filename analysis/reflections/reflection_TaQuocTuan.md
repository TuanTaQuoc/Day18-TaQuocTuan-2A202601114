# Individual Reflection - Lab 18

**Tên:** Tạ Quốc Tuấn  
**MSSV:** 2A202601114  
**Module phụ trách:** M1, M2, M3, M4, M5

---

## 1. Mapping bài giảng vào code

| Lecture Concept | Module | Hàm cụ thể | Observation |
|----------------|--------|------------|-------------|
| Semantic chunking | M1 | `chunk_semantic()` | Nhóm câu theo similarity; có fallback paragraph khi embedding model không khả dụng. |
| Hierarchical chunking | M1 | `chunk_hierarchical()` | Tách parent/child để retrieve child nhưng vẫn giữ `parent_id` cho context rộng hơn. |
| BM25 + Dense fusion | M2 | `BM25Search`, `DenseSearch`, `reciprocal_rank_fusion()` | RRF giúp hợp nhất kết quả lexical và semantic, production đạt Context Precision 0.9333. |
| Cross-encoder reranking | M3 | `CrossEncoderReranker.rerank()` | Model ban đầu `BAAI/bge-reranker-v2-m3` crash trên Windows; đổi sang `cross-encoder/ms-marco-MiniLM-L-6-v2` để test pass. |
| RAGAS 4 metrics | M4 | `evaluate_ragas()` | Production: faithfulness 0.5975, answer_relevancy 0.6312, context_precision 0.9333, context_recall 0.7417. |
| Contextual enrichment | M5 | `_enrich_single_call()`, `enrich_chunks()` | Combined mode làm giàu chunk bằng context, summary, questions và metadata trong 1 call/chunk. |

## 2. Khó khăn & Cách giải quyết

- **Lỗi lớn nhất:** `Windows fatal exception: access violation` khi chạy `pytest tests/test_m3.py -v`.
- **Vị trí lỗi:** `src/m3_rerank.py`, tại `CrossEncoder(self.model_name)` khi load `BAAI/bge-reranker-v2-m3`.
- **Cách debug:** Chạy riêng M3 để cô lập lỗi, đọc stack trace thấy crash nằm trong `torch`/`transformers`, sau đó đổi sang model nhẹ hơn `cross-encoder/ms-marco-MiniLM-L-6-v2`.
- **Kết quả:** M3 pass 5/5, toàn bộ test pass 37/37.
- **Khó khăn khi chạy pipeline:** `python main.py`/pipeline trên Windows có thể dừng vì encoding console hoặc thời gian chạy dài. Chạy `src/pipeline.py` với UTF-8 đã sinh được `ragas_report.json`.

## 3. Kết quả & Insight

Production pipeline đạt precision cao hơn baseline một chút, nhưng faithfulness và recall thấp hơn. Các failure chính xuất hiện ở câu hỏi cần tổng hợp nhiều tài liệu hoặc cần chọn đúng policy hiện hành, ví dụ nghỉ phép v2024 thay vì v2023. Điều này cho thấy RAG không chỉ cần retrieval mạnh mà còn cần metadata version, reranking có awareness về tài liệu mới/cũ, và prompt kiểm tra conflict.

## 4. Action Plan cho project cá nhân

### Project: Vietnamese Internal Policy Assistant

### Hiện tại
- RAG pipeline hiện tại: load tài liệu nội bộ, chunk, hybrid search, rerank, generate answer và đánh giá bằng RAGAS.
- Known issues: dễ chọn nhầm chính sách cũ, câu hỏi multi-hop thiếu context, và answer đôi khi abstain "Không tìm thấy" dù ground truth tồn tại.

### Plan áp dụng
1. [ ] Chunking strategy: dùng hierarchical chunking làm mặc định để child retrieval chính xác nhưng vẫn có parent context khi trả lời.
2. [ ] Search: dùng hybrid BM25 + Dense + RRF để cân bằng keyword tiếng Việt và semantic matching.
3. [ ] Reranking: dùng CrossEncoder nhẹ, ổn định trên Windows; tăng candidate trước rerank cho câu hỏi multi-hop.
4. [ ] Evaluation: dùng RAGAS định kỳ với 4 metrics, đặc biệt theo dõi context_recall và faithfulness.
5. [ ] Enrichment: thêm metadata `category`, `version`, `status`, `effective_date` để xử lý tài liệu có nhiều phiên bản.

### Timeline
- Tuần 1: Chuẩn hóa metadata cho tài liệu policy, đánh dấu tài liệu hiện hành và tài liệu cũ.
- Tuần 2: Cải thiện retrieval/rerank cho câu hỏi nhiều ý và câu hỏi có version conflict.
- Tuần 3: Chạy RAGAS, phân tích bottom failures, tối ưu prompt answer.
- Tuần 4: Đóng gói pipeline, viết hướng dẫn chạy và checklist kiểm thử trước khi deploy.

## 5. Tự đánh giá

| Tiêu chí | Tự chấm (1-5) |
|----------|---------------|
| Hiểu bài giảng | 4 |
| Code quality | 4 |
| Teamwork | 4 |
| Problem solving | 5 |
