# Enhance sequence — Nới error budget circuit breaker tạm thời trong lúc rolling restart

Đây là flow **enhance** hoàn toàn mới so với base, đáp ứng **yêu cầu 4** của đề bài: khi rolling restart đang diễn ra, một vài request timeout do đúng lúc instance đang drain là chuyện bình thường và có thể dự đoán trước — circuit breaker phải tạm nới ngưỡng chịu lỗi trong cửa sổ thời gian đó, tránh ngắt toàn bộ traffic của service chỉ vì vài lỗi tạm thời.

```mermaid
sequenceDiagram
    actor Ops as Đội vận hành
    participant GW as API Gateway
    participant CB as Circuit Breaker (service X)
    participant RR as ROLLING_RESTART_RUN

    Ops->>RR: Bắt đầu rolling restart service X
    RR->>CB: Thông báo rolling restart đang chạy
    CB->>CB: UPDATE error_budget_multiplier = 3.0, widened_until = now + 10 phút
    Note over CB: Ngưỡng lỗi cho phép trong cửa sổ này tăng gấp 3 lần mức bình thường, chỉ áp dụng tạm thời

    loop Trong lúc rolling restart
        GW->>CB: Ghi nhận kết quả mỗi request (success/timeout)
        alt Tỷ lệ lỗi vẫn dưới ngưỡng đã nới rộng
            CB->>CB: state vẫn = closed, tiếp tục cho traffic đi qua bình thường
        else Tỷ lệ lỗi vượt cả ngưỡng đã nới rộng (có bug thật, không phải do drain)
            CB->>CB: state = open, ngắt traffic tới service X
            CB-->>GW: Từ chối route request mới tới service X
        end
    end

    RR->>CB: Rolling restart hoàn tất
    CB->>CB: UPDATE error_budget_multiplier = 1.0 (khôi phục ngưỡng bình thường)
```
