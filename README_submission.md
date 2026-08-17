# Báo Cáo Lab 17: Zep Memory Agent

## 1. Các câu hỏi lý thuyết (Mục 5.2)

**Layer quan trọng nhất trong bộ test này:** `Long-term` memory (ví dụ ở các case E02, E03, E08, E09). Nó giúp Agent theo dõi các "Open Loop" như deadline (`LAB-REPORT-1600`) và "Preference" về ngôn ngữ lập trình của user qua nhiều session, tạo ra trải nghiệm cá nhân hóa.

**Trade-off Context Block (Zep) vs Redis + Qdrant tự build:**
Sử dụng Zep giúp tiết kiệm công sức tổng hợp ngữ cảnh (fact extraction, graph entities, user summaries) nhờ Context Block API tự động, trong khi tự build bằng Redis+Qdrant đòi hỏi phải code orchestration phức tạp. Đổi lại, Zep tạo ra sự phụ thuộc vào Third-party API, có giới hạn về token (rate limits) và khó tinh chỉnh (black-box) so với hệ thống tự xây dựng toàn quyền.

**Guardrail chống memory poisoning:**
Cần phân quyền truy cập chặt chẽ (`user_id` scoping) như thực hiện trong Zep để đảm bảo user này không ảnh hưởng đến memory của user khác (user isolation - E09). Ngoài ra, cơ chế Heartbeat chỉ cho phép tổng hợp (recap) hoặc ghi chú các thay đổi trạng thái thay vì cấp quyền ghi instruction trực tiếp từ input của user vào hệ thống, tránh prompt injection tấn công thẳng vào memory.

## 2. Phân tích kết quả Benchmark (Mục 4.5)

1. **Layer có hit rate thấp nhất:** Trong bản triển khai này, tất cả các layer đều đạt 100% (11/11). Tuy nhiên ở chế độ `no_memory`, `episodic` và `semantic` có hit rate 0% vì mọi context ngoài luồng ngắn hạn đều bị mất.
2. **Query có số lượng token được retrieve nhiều nhất:** Case `E03` (Long-term open loop) lấy về **903 tokens** do kéo theo toàn bộ User Summary, Facts, Entities và Threads metadata.
3. **Phân tích Case mixed (E07):** Case E07 bắt buộc kết hợp cả **Long-term memory** (để lấy preference ngôn ngữ "Python" của Minh) và **Semantic memory** (để lấy "Idempotency-Key" trong quy tắc retry payment). Do đó evidence bắt buộc là cả "Python" và "Idempotency-Key".
4. **Token reduction:** Chế độ `no_memory` có độ giảm token (token reduction) cao tới 81.8% vì hệ thống hoàn toàn loại bỏ việc kéo ngữ cảnh mở rộng, khiến dung lượng input vào model rất nhỏ. Tuy nhiên điều này khiến Hit Rate giảm mạnh xuống chỉ còn 18.2% do Agent "mất trí nhớ" về các kiến thức domain và facts quá khứ.

## 3. Minh chứng Privacy Drill
Quá trình xoá người dùng `minh-lab17` và xác minh bằng lệnh `verify-only` đã được thực thi và in ra kết quả như sau:

```
Deleting user-scoped memory for 'minh-lab17'...
Redis keys deleted: 0
Zep user absent: True
Redis user keys remaining: 0
Shared semantic KB remains intact because it stores domain knowledge, not user PII.
```
