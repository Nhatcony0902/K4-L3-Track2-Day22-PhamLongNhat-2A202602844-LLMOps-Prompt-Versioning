# Evidence — Day 22: LangSmith + Prompt Versioning

**Học viên:** Phạm Long Nhật — 2A202602844
**Provider:** OpenAI `gpt-4o-mini` + `text-embedding-3-small` · **LangSmith project:** `day22-lab`

## Danh sách tệp

| Tệp | Nội dung |
|---|---|
| `01_langsmith_traces.png` | Danh sách traces trong project `day22-lab` (400 traces trong 1 ngày) |
| `01_langsmith_project_stats.png` | Thống kê project: 401 traces, error rate 0%, P50 1.48s |
| `02_prompt_hub.png` | 2 prompt `pham-long-nhat-rag-prompt-v1` / `-v2` trên Prompt Hub (email đã che) |
| `02_ab_routing_log.txt` | Log A/B routing: pull cả 2 prompt từ Hub, nhãn `[prompt-v1]` / `[prompt-v2]` cho từng câu |
| `03_ragas_scores.png` | Bảng so sánh RAGAS V1 vs V2 |
| `03_ragas_report.json` | Bản sao `data/ragas_report.json` |
| `04_pii_demo_log.txt` | 7 test case PII (email, phone, SSN, thẻ tín dụng, nhiều loại cùng lúc, đầu vào sạch) |
| `04_json_demo_log.txt` | 6 test case JSON (hợp lệ, fences, nháy đơn, dấu phẩy thừa, lỗi kết hợp, không sửa được) |

Số traces: Bước 1 tạo 50 trace `rag-query`; Bước 2 chạy 3 lần → 150 trace `ab-rag-query`; phần còn lại là các run của RAGAS evaluation.

## Kết quả RAGAS (50 cặp QA × 2 phiên bản)

| Metric | V1 (ngắn gọn) | V2 (chuyên gia, có cấu trúc) | Tốt hơn |
|---|---|---|---|
| faithfulness | **0.9710** | 0.9518 | V1 |
| answer_relevancy | **0.9184** | 0.8929 | V1 |
| context_recall | 1.0000 | 1.0000 | Hòa |
| context_precision | 0.9383 | **0.9417** | ≈ Hòa |

Cả 2 phiên bản đều đạt faithfulness ≥ 0.9.

## Phân tích: vì sao V1 nhỉnh hơn V2

Hai prompt dùng chung retriever (FAISS, chunk 500/overlap 50, k=3) và chung LLM; chỉ khác system prompt:

- **V1** — "trợ lý thân thiện, trả lời 2-4 câu, chỉ dựa trên context".
- **V2** — "chuyên gia phân tích, xác định facts rồi viết 3-5 câu: câu đầu trả lời trực tiếp, các câu sau bổ sung chi tiết".

Đo trên traces của Bước 2: câu trả lời V1 trung bình **37 từ / ~1.8 câu**, V2 trung bình **69 từ / ~3.2 câu** — V2 dài gần gấp đôi.

1. **Faithfulness (0.971 vs 0.952).** RAGAS tách câu trả lời thành các claim rồi kiểm tra từng claim có được context hỗ trợ không. V2 bị yêu cầu "bổ sung chi tiết" nên sinh nhiều claim hơn; mỗi claim thêm là một cơ hội để model diễn giải, khái quát hoá hoặc chèn kiến thức nền không có nguyên văn trong 3 chunk → tỉ lệ claim được hỗ trợ giảm nhẹ. V1 ngắn, gần như chỉ lặp lại fact chính của context nên ít claim "thừa".
2. **Answer relevancy (0.918 vs 0.893).** Metric này sinh ngược câu hỏi từ câu trả lời rồi so embedding với câu hỏi gốc. Câu trả lời V2 có thêm chi tiết phụ (ví dụ, hệ quả, ngữ cảnh) nên câu hỏi sinh ngược "rộng" hơn câu hỏi gốc → độ tương đồng thấp hơn. Câu trả lời ngắn và trúng đích của V1 ánh xạ ngược sát hơn.
3. **Context recall / context precision (gần như bằng nhau).** Hai metric này chỉ so `retrieved_contexts` với `reference`, không phụ thuộc câu trả lời. Vì V1 và V2 dùng cùng retriever nên contexts giống hệt nhau → recall bằng nhau (1.0); chênh lệch 0.003 ở precision là nhiễu của LLM-judge, không phải khác biệt thật giữa 2 prompt.

**Kết luận:** với knowledge base dạng định nghĩa ngắn và bộ câu hỏi factual như lab này, prompt ngắn gọn (V1) an toàn hơn về độ trung thực và độ liên quan. V2 phù hợp hơn khi người dùng cần giải thích dài, nhưng nên ràng buộc chặt hơn (ví dụ "mỗi câu phải dẫn được về một đoạn context") để giữ faithfulness.

## Ghi chú

- Khi chấm V1 có 2 lần `OpenAIConnectionError` (lỗi mạng tạm thời) → 1 sample thiếu điểm `answer_relevancy` và `context_recall`; trung bình của 2 metric này ở V1 tính trên 49 sample, các metric còn lại đủ 50.
- Log có nhiều dòng `LLM returned 1 generations instead of requested 3` — cảnh báo bình thường của RAGAS với OpenAI, không ảnh hưởng kết quả.
- A/B routing dùng MD5 của `request_id` (không dùng `hash()` built-in vì Python random hoá seed giữa các tiến trình); 3 lần chạy Bước 2 cho cùng phân bố V1=19 / V2=31.
