# Sequence Diagram — Enhance: Retry Failed Transcode

Đây là **enhance**, flow mới xử lý yêu cầu "nếu job transcode fail giữa chừng, phải retry đúng phần việc còn thiếu, không transcode lại từ đầu toàn bộ video".

```mermaid
sequenceDiagram
    participant Transcoder as Transcode Worker
    participant VideoSvc as Video Service
    participant Storage as Video Storage
    participant RetryQueue as Retry Queue

    Transcoder->>Storage: Transcode rendition theo từng segment
    Transcoder--xVideoSvc: Worker crash / hết dung lượng tạm giữa chừng

    VideoSvc->>Storage: Kiểm tra segment nào của RENDITION đã transcode xong
    Storage-->>VideoSvc: Danh sách segment done, failed_segment_ref cho phần còn thiếu

    VideoSvc->>RetryQueue: Enqueue retry job chỉ cho phần segment còn thiếu
    RetryQueue->>Transcoder: Dispatch worker mới, tiếp tục từ failed_segment_ref
    Transcoder->>Storage: Transcode nốt phần còn thiếu
    Transcoder-->>VideoSvc: Rendition hoàn chỉnh, không phải làm lại từ đầu
```
