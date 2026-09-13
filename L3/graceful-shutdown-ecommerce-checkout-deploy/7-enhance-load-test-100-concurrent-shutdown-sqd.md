# Enhance sequence — Test toàn trình 100 request đồng thời rồi trigger shutdown giữa chừng

Đây là flow **enhance** hoàn toàn mới so với base, đáp ứng **yêu cầu 5** của đề bài: gửi 100 request checkout đồng thời rồi trigger shutdown giữa chừng, verify không có request nào bị mất kết nối nửa đường — mỗi request phải nhận được response (kể cả response lỗi rõ ràng như 503/failed_incomplete), không bao giờ là timeout im lặng.

```mermaid
sequenceDiagram
    participant Test as Test Orchestrator
    participant LoadGen as Load Generator
    participant LB as Load Balancer
    participant App as Checkout Instance A
    participant Run as CHECKOUT_LOAD_TEST_RUN

    Test->>Run: INSERT CHECKOUT_LOAD_TEST_RUN(total_requests=100, status=running)
    Test->>LoadGen: Bắn 100 request checkout đồng thời qua LB tới App
    LoadGen->>LB: 100 request song song
    LB->>App: Route cả 100 request (App đang readiness=true)

    Note over Test: Giữa lúc 100 request đang xử lý dở (một số đã xong, một số đang gọi Gateway), Test trigger SIGTERM cho App
    Test->>App: Gửi SIGTERM
    App->>LB: Báo readiness=false ngay lập tức
    LB->>LB: Ngừng route request mới, 100 request đang chạy dở không bị huỷ

    loop Với từng request trong 100 request đang xử lý dở
        alt Xử lý xong trước khi hết grace period
            App-->>LoadGen: Response 200 (thành công)
        else Hết grace period mà chưa xong
            App-->>LoadGen: Response 503 kèm order failed_incomplete (lỗi rõ ràng, không phải timeout im lặng)
        end
    end

    LoadGen->>Test: Tổng hợp: responses_received=100, silent_timeouts=0
    Test->>Run: UPDATE responses_received=100, silent_timeouts=0

    alt silent_timeouts == 0 và mọi request đều có response
        Test->>Run: UPDATE status=passed
    else Có request không nhận được response nào (timeout im lặng)
        Test->>Run: UPDATE status=failed
    end
```
