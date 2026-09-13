# Enhance sequence — Đo lường mức giảm DB query trùng lặp so với baseline

Đây là **enhance**, flow hoàn toàn mới mô tả cách hệ thống đo lường hiệu quả của coordinator sau khi triển khai. Đáp ứng yêu cầu 4: cần con số cụ thể chứng minh giảm được >90% query trùng lặp trong 1 giây đầu sau cache miss, dựa trên dữ liệu `CACHE_MISS_EVENT` được ghi lại ở flow acquire-lock.

```mermaid
sequenceDiagram
    participant App as Application Instances
    participant DB2 as CACHE_MISS_EVENT store
    participant Job as Metrics Aggregator Job
    actor SRE as SRE/Dev xem dashboard

    Note over App,DB2: Trong flow read-article-cache-miss, mỗi request đều ghi 1 CACHE_MISS_EVENT (role, action, wait_time_ms)

    App->>DB2: INSERT CACHE_MISS_EVENT (role=leader, action=queried_db)
    App->>DB2: INSERT CACHE_MISS_EVENT (role=follower, action=waited_lock)
    App->>DB2: INSERT CACHE_MISS_EVENT (role=follower, action=served_stale)

    loop Mỗi phút
        Job->>DB2: SELECT cache_key, COUNT(*) FILTER (action=queried_db) AS actual_db_queries, COUNT(*) AS total_requests GROUP BY cache_key trong cửa sổ 1s sau miss đầu tiên
        DB2-->>Job: Kết quả tổng hợp theo từng đợt cache miss

        Job->>Job: Tính reduction_rate = 1 - (actual_db_queries / total_requests_ước_tính_nếu_không_có_coordinator)
    end

    Job-->>SRE: Publish dashboard, vd "Đợt cache miss bài viết 123: 420 request, chỉ 1 query DB thật, giảm 99.7% so với baseline"

    Note over Job,SRE: Baseline tham chiếu là số liệu đo được ở flow base (mỗi request 1 query), so sánh trực tiếp để xác nhận đạt mục tiêu giảm >90%
```
