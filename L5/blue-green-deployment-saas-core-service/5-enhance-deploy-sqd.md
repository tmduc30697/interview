# Enhance sequence — Deploy (blue-green, smoke test, migration backward-compatible)

Đây là **enhance**, cùng flow "Deploy" đã có ở base nhưng nay thay đổi hoàn toàn theo yêu cầu 1-3 của đề bài: green được deploy song song (không tắt blue), migration phải backward-compatible để blue vẫn chạy đúng, smoke test bắt buộc pass trước khi router chuyển traffic thật, và việc chuyển traffic là thao tác tức thời ở router.

```mermaid
sequenceDiagram
    participant CI as CI/CD
    participant Blue as Blue Deployment (đang live)
    participant Green as Green Deployment (mới)
    participant DB as Shared Database
    participant Smoke as Smoke Test Runner
    participant Router as Router/Load Balancer

    CI->>DB: Áp dụng DB_MIGRATION (backward_compatible=true, phase=expand)
    Note over DB: Blue vẫn tiếp tục hoạt động đúng với schema đã đổi trong lúc chuyển tiếp
    CI->>Green: Deploy phiên bản mới thành GREEN, song song với Blue (Blue không bị tắt)
    Green->>Green: status=staging → smoke_testing
    CI->>Smoke: Gọi các endpoint quan trọng lên Green
    Smoke->>DB: Ghi SMOKE_TEST_RUN (test_name, result)
    alt Smoke test fail
        Smoke-->>CI: result=fail
        CI->>Green: Huỷ Green, giữ nguyên Blue đang phục vụ
        CI-->>CI: Deploy bị dừng, không chuyển traffic
    else Smoke test pass
        Smoke-->>CI: result=pass
        Green->>Green: status=live
        CI->>Router: Tạo ROUTER_SWITCH_EVENT (from=Blue, to=Green, reversible_until=+vài giây)
        Router->>Router: Đổi target gần như tức thời — client không cần làm gì
        Router-->>Blue: Ngừng gửi traffic MỚI tới Blue (connection cũ xử lý ở flow "Connection draining")
        Note over Router: Có thể đảo ngược switch trong vài giây tiếp theo (xem flow "Rollback")
    end
```
