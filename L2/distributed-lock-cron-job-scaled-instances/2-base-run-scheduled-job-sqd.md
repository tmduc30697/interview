# Base sequence — Mỗi instance tự chạy job theo lịch riêng, chạy trùng lặp

Đây là **base**, flow chạy job định kỳ hiện tại: mỗi instance/pod có timer nội bộ riêng theo `cron_expression`, khi tới giờ thì tự chạy job mà không hỏi ý kiến ai. Đây chính là vấn đề nêu trong đề bài — service được scale ra nhiều instance để chịu tải, nhưng job như gửi báo cáo hay dọn dữ liệu lại vô tình chạy nhiều lần trùng lặp mỗi chu kỳ, có thể gây gửi báo cáo trùng hoặc dọn dữ liệu race nhau.

```mermaid
sequenceDiagram
    participant PodA as Instance/Pod A
    participant PodB as Instance/Pod B
    participant PodC as Instance/Pod C
    participant DB as Database / External System

    Note over PodA,PodC: Cả 3 pod đều có cùng cron_expression, timer nội bộ kích hoạt cùng lúc

    par Cả 3 instance cùng trigger job đồng thời
        PodA->>PodA: Cron trigger "send_daily_report"
        PodB->>PodB: Cron trigger "send_daily_report"
        PodC->>PodC: Cron trigger "send_daily_report"
    end

    par Không ai biết ai cũng đang chạy, cả 3 đều thực thi
        PodA->>DB: Chạy job, gửi báo cáo, ghi JOB_RUN
        PodB->>DB: Chạy job, gửi báo cáo, ghi JOB_RUN
        PodC->>DB: Chạy job, gửi báo cáo, ghi JOB_RUN
    end

    Note over DB: Báo cáo bị gửi 3 lần trùng lặp, hoặc dữ liệu bị dọn 3 lần race nhau
```
