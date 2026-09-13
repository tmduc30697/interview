# Enhance sequence — Admin revoke permission (broadcast + ack + audit)

Đây là **enhance**, cùng flow "Admin revoke permission" đã có ở base nhưng nay thay đổi hoàn toàn theo yêu cầu 1, 2 và 5 của đề bài: broadcast invalidation tới toàn bộ instance qua pub/sub, chờ xác nhận trước khi coi revoke đã có hiệu lực đầy đủ, và log lại toàn bộ cho audit.

```mermaid
sequenceDiagram
    actor Admin
    participant AdminInstance as Service Instance xử lý request admin
    participant DB as USER_PERMISSION store
    participant PubSub as Pub/Sub broadcast
    participant InstanceA as Instance A
    participant InstanceB as Instance B
    participant Audit as PERMISSION_INVALIDATION_AUDIT_LOG store

    Admin->>AdminInstance: Revoke quyền admin của user X
    AdminInstance->>DB: Cập nhật USER_PERMISSION (xoá quyền)
    DB-->>AdminInstance: Cập nhật thành công
    AdminInstance->>PubSub: Publish PERMISSION_INVALIDATION_EVENT (user_id=X, revoked_permission=admin)
    par Broadcast tới toàn bộ instance
        PubSub->>InstanceA: Nhận event, invalidate PERMISSION_CACHE_ENTRY(user_id=X) cục bộ
        InstanceA-->>PubSub: Gửi PERMISSION_INVALIDATION_ACK (status=acked)
    and
        PubSub->>InstanceB: Nhận event, invalidate PERMISSION_CACHE_ENTRY(user_id=X) cục bộ
        InstanceB-->>PubSub: Gửi PERMISSION_INVALIDATION_ACK (status=acked)
    end
    PubSub-->>AdminInstance: Tổng hợp ACK từ toàn bộ instance đã biết
    alt Toàn bộ instance đã ack
        AdminInstance->>AdminInstance: PERMISSION_INVALIDATION_EVENT status=confirmed
        AdminInstance->>Audit: Ghi PERMISSION_INVALIDATION_AUDIT_LOG (admin, user, permission, thời điểm, broadcast_summary=full)
        AdminInstance-->>Admin: "Revoke đã có hiệu lực trên toàn bộ instance"
    else Có instance không ack (mất kết nối tạm thời)
        AdminInstance->>AdminInstance: PERMISSION_INVALIDATION_EVENT status=partial
        AdminInstance->>Audit: Ghi log kèm rõ instance nào không ack
        Note over AdminInstance: Instance chưa ack vẫn được bảo vệ bởi ttl_seconds rất ngắn (xem flow "Authorize request") — cửa sổ rủi ro bị giới hạn tối đa
        AdminInstance-->>Admin: "Revoke có hiệu lực ngay ở đa số instance, 1 instance đang chờ xác nhận"
    end
```
