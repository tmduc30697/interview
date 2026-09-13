# Enhance sequence — Invalidation failure & retry

Đây là **enhance**, flow hoàn toàn mới, phát sinh từ enhance — chưa tồn tại ở base (ở base, purge CDN fire-and-forget, không ai biết nó có thất bại hay không). Đáp ứng yêu cầu thứ 4 của đề bài: khi 1 tầng invalidate thất bại (vd API purge CDN timeout), phải retry có theo dõi trạng thái và log rõ tầng nào thành công/thất bại.

```mermaid
sequenceDiagram
    participant Job as INVALIDATION_JOB store
    participant Step as INVALIDATION_STEP store
    participant CDN as CDN Edge API
    participant Ops as Ops Dashboard / Alerting

    Note over Step: INVALIDATION_STEP(layer=cdn_edge) vừa fail (API timeout hoặc trả lỗi)
    Step->>Step: status=retrying, attempt_count += 1
    Job->>Ops: Log ngay: "layer=cdn_edge failed, attempt=1, các layer khác vẫn giữ trạng thái success"

    loop Retry với backoff, tối đa N lần
        Step->>CDN: Gọi lại API purge
        alt Thành công
            CDN-->>Step: Xác nhận đủ PoP
            Step->>Step: status=success, completed_at=now
            Job->>Job: Kiểm tra toàn bộ INVALIDATION_STEP → nếu đủ 3 tầng success, INVALIDATION_JOB status=completed
            Job->>Ops: Log "layer=cdn_edge recovered sau retry"
        else Vẫn thất bại
            CDN-->>Step: Timeout/lỗi tiếp
            Step->>Step: attempt_count += 1
        end
    end

    alt Hết số lần retry cho phép mà vẫn fail
        Step->>Step: status=failed (dừng retry tự động)
        Job->>Job: INVALIDATION_JOB status=partially_stale
        Job->>Ops: Cảnh báo rõ ràng: "article X — query_cache: success, app_cache: success, cdn_edge: failed — vẫn còn nội dung cũ ở CDN"
        Ops->>Ops: Vận hành biết chính xác tầng nào đang stale để xử lý thủ công thay vì tưởng đã invalidate hết
    end
```
