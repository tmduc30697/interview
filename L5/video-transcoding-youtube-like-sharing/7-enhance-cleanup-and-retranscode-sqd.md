# Sequence Diagram — Enhance: Cleanup and Retranscode Original

Đây là **enhance**, flow mới xử lý yêu cầu dọn file gốc/file tạm sau X ngày để tối ưu chi phí lưu trữ, nhưng vẫn giữ khả năng re-transcode nếu cần đổi codec sau này.

```mermaid
sequenceDiagram
    participant Scheduler as Retention Scheduler
    participant VideoSvc as Video Service
    participant Storage as Video Storage
    actor Admin

    Scheduler->>VideoSvc: Check RETENTION_POLICY, video quá retain_days
    VideoSvc->>Storage: Xác nhận đã có đủ RENDITION cần thiết (đã publish ổn định)

    alt đủ điều kiện dọn
        VideoSvc->>Storage: Xóa raw file gốc, giữ lại các RENDITION đã transcode
        VideoSvc->>VideoSvc: Set original_cleaned_up = true
    end

    Note over VideoSvc,Storage: Sau này nếu cần đổi codec

    Admin->>VideoSvc: Yêu cầu re-transcode video (đổi codec mới)
    VideoSvc->>Storage: Kiểm tra retranscode_possible

    alt raw gốc đã xóa
        VideoSvc->>Storage: Dùng RENDITION chất lượng cao nhất còn lại làm nguồn để re-transcode
        Note over VideoSvc: Chất lượng re-transcode giới hạn bởi rendition cao nhất còn giữ lại
    else raw gốc còn giữ
        VideoSvc->>Storage: Re-transcode trực tiếp từ raw file gốc
    end
    VideoSvc-->>Admin: Re-transcode hoàn tất
```
