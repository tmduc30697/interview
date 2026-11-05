# SaaS gọi LLM/AI provider bên thứ ba cho tính năng chat AI

**Hệ thống:** Một SaaS có tính năng chat AI gọi API của nhà cung cấp mô hình ngôn ngữ bên ngoài để trả lời user.

**Vai trò của flow:** Circuit breaker + retry giúp xử lý khi provider AI bị rate limit, timeout, hoặc quá tải, tránh trải nghiệm user bị treo vô thời hạn hoặc gọi lại làm tốn quota/tiền không cần thiết.

**Yêu cầu cụ thể:**
- Retry chỉ thực hiện với lỗi timeout/5xx và tối đa số lần retry giới hạn (ví dụ 2 lần) với backoff, vì mỗi lần gọi provider AI có chi phí (tính theo token) — retry vô tội vạ gây tốn tiền.
- Khi provider trả lỗi 429 (rate limit), phải đọc header retry-after (nếu có) để backoff đúng thời gian được yêu cầu, không tự đoán.
- Circuit breaker mở khi tỉ lệ lỗi/timeout của provider vượt ngưỡng, và hệ thống phải có fallback rõ ràng khi breaker mở (ví dụ chuyển sang provider dự phòng thứ hai, hoặc trả thông báo lỗi thân thiện cho user thay vì treo loading vô hạn).
- Phải phân biệt và không retry với lỗi do chính request của user sai (ví dụ input vượt quá giới hạn token, prompt bị filter) — retry những lỗi này chỉ tốn thời gian mà không giải quyết được gì.
- Đo lường: latency thêm vào do retry (retry overhead), tỉ lệ request phải fallback sang provider dự phòng, và chi phí phát sinh do retry để đưa vào theo dõi ngân sách AI.
