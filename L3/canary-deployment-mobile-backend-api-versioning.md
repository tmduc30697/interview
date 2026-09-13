# Canary rollout cho backend mobile phải serving song song nhiều phiên bản app cũ

**Hệ thống:** Một backend phục vụ ứng dụng mobile, trong đó người dùng không update app đồng loạt — tại một thời điểm, backend phải phục vụ đồng thời nhiều phiên bản app khác nhau đang cài trên máy người dùng, có thể chênh nhau nhiều tháng phát hành.

**Vai trò của flow:** Canary deployment ở đây không chỉ kiểm tra phiên bản backend mới có lỗi hay không, mà còn phải đảm bảo phiên bản mới không phá vỡ hợp đồng API mà các phiên bản app cũ đang phụ thuộc, vì không thể ép người dùng update app ngay lập tức.

**Yêu cầu cụ thể:**
- Trước khi canary, phải xác định rõ tập các phiên bản app cũ tối thiểu còn cần hỗ trợ (dựa trên số liệu người dùng thực tế đang dùng phiên bản nào) và test hợp đồng API mới với từng phiên bản đó, không chỉ test với app mới nhất.
- Traffic canary phải được phân chia sao cho bao gồm cả request từ app phiên bản cũ lẫn mới, không chỉ dồn canary vào nhóm user đã update app mới nhất — nếu không, lỗi tương thích ngược chỉ lộ ra sau khi đã mở rộng traffic lên cao.
- Xử lý trường hợp phiên bản backend mới đổi định dạng response (thêm field bắt buộc, đổi kiểu dữ liệu) khiến app cũ không parse được — phải phát hiện qua tỷ lệ crash/lỗi parse tăng bất thường ở nhóm client cũ trong lúc canary, tách biệt với lỗi kỹ thuật chung của backend.
- Đảm bảo cơ chế theo dõi lỗi trong canary phân tách được theo phiên bản app gọi tới (thông qua header version của client), để biết chính xác lỗi phát sinh ở nhóm client nào thay vì gộp chung tỷ lệ lỗi toàn bộ traffic, tránh bỏ sót lỗi chỉ ảnh hưởng một nhóm phiên bản cũ thiểu số.
- Định nghĩa chiến lược rollback không đơn giản là "quay lại code cũ" nếu phiên bản mới đã bắt đầu ghi dữ liệu theo schema mới (ví dụ trường mở rộng cho tính năng mới) — dữ liệu đó vẫn phải đọc được bởi phiên bản cũ đang phục vụ các client legacy sau khi rollback, tránh gãy tương thích ngược thêm lần nữa.

---

# Base ERD — Canary backend mobile trước khi nhận diện đa phiên bản app

Đây là **base**: mô hình dữ liệu suy luận từ ngữ cảnh đề bài, thể hiện trạng thái hệ thống **trước khi** canary nhận diện đa phiên bản app client. Đề bài nói backend đã phục vụ nhiều phiên bản app cũ và có sẵn khái niệm canary — nên base cần đủ: phiên bản app client, backend deployment (đã có role stable/canary), test hợp đồng API (chỉ test với app mới nhất), và log request (có ghi app_client_version_header nhưng chưa dùng để tách metric). Chưa có entity nào phục vụ stratify traffic theo version/metric tách theo version/kế hoạch rollback tương thích ngược — những thứ đó là phần enhance.

```mermaid
erDiagram
    BACKEND_DEPLOYMENT ||--o{ API_CONTRACT_TEST : "tested by"
    BACKEND_DEPLOYMENT ||--o{ API_REQUEST_LOG : serves
    BACKEND_DEPLOYMENT ||--o{ ERROR_METRIC_WINDOW : measured

    APP_CLIENT_VERSION {
        string id PK
        string version
        int active_user_count
    }
    BACKEND_DEPLOYMENT {
        string id PK
        string version
        string role "stable | canary"
        int traffic_percent
    }
    API_CONTRACT_TEST {
        string id PK
        string backend_version FK
        string tested_against_app_version
        string result
    }
    API_REQUEST_LOG {
        string id PK
        string backend_version FK
        string app_client_version_header
        int status_code
        boolean parse_error
        datetime created_at
    }
    ERROR_METRIC_WINDOW {
        string id PK
        string backend_version FK
        datetime window_start
        datetime window_end
        decimal error_rate "gộp chung mọi client version"
    }
```

---

# Base sequence — Pre-release contract test (chỉ test app mới nhất)

Đây là **base**, flow "Test hợp đồng API trước khi canary" ở trạng thái hiện tại — chỉ test với phiên bản app mới nhất. Flow này liên quan mật thiết tới enhance vì yêu cầu thứ 1 của đề bài chính là sửa đúng lỗ hổng này.

```mermaid
sequenceDiagram
    actor Backend as Backend Engineer
    participant CI as CI/CD
    participant Test as API_CONTRACT_TEST runner
    participant LatestApp as App version mới nhất

    Backend->>CI: Chuẩn bị release backend version mới
    CI->>Test: Chạy contract test với LatestApp
    Test->>LatestApp: Gọi API theo hợp đồng hiện tại
    LatestApp-->>Test: Parse response thành công
    Test-->>CI: result=pass
    CI-->>Backend: "Contract test pass, sẵn sàng canary"
    Note over Test,LatestApp: Không test với bất kỳ phiên bản app cũ nào đang được người dùng thực tế sử dụng — nếu backend đổi format response, app cũ có thể parse sai mà không ai biết trước khi canary
```

---

# Base sequence — Canary rollout & rollback (chưa nhận diện version app)

Đây là **base**, flow "Canary rollout" ở trạng thái hiện tại — traffic canary chọn ngẫu nhiên không phân biệt phiên bản app, metric lỗi gộp chung, rollback chỉ đơn giản là revert code. Flow này liên quan mật thiết tới enhance vì yêu cầu 2, 3, 4 và 5 của đề bài đều nhằm sửa đúng các lỗ hổng ở đây.

```mermaid
sequenceDiagram
    participant Router as Router
    participant Canary as Backend canary
    participant Stable as Backend stable
    participant Metric as ERROR_METRIC_WINDOW store
    actor OnCall as On-call Engineer
    actor NewApp as Client app mới nhất
    actor OldApp as Client app phiên bản cũ

    Router->>Router: Random 5% traffic vào canary, không xét app_client_version_header
    Note over Router: Đa số user đã update app mới nên ngẫu nhiên dễ dồn canary vào nhóm NewApp, ít chạm tới OldApp
    NewApp->>Canary: Request (thường xuyên rơi vào canary)
    OldApp->>Stable: Request (hiếm khi rơi vào canary do ngẫu nhiên thiên lệch)
    Canary->>Metric: Ghi error_rate gộp chung mọi client version
    Metric-->>OnCall: error_rate tổng thể vẫn thấp (vì OldApp ít vào canary nên lỗi parse ở nhóm này chưa lộ ra)
    OnCall->>Router: Thấy ổn, tiếp tục tăng traffic canary
    Note over Canary,OldApp: Khi traffic tăng cao hơn, OldApp mới bắt đầu dính nhiều vào canary và lộ ra lỗi parse response — phát hiện quá muộn
    OnCall->>Canary: Nếu phát hiện lỗi, rollback = deploy lại code backend cũ
    Note over Canary,Stable: Nếu canary đã ghi dữ liệu theo schema mới trước khi rollback, code cũ có thể không đọc được — chưa có kế hoạch xử lý
```

---

# Enhance ERD — sau khi canary nhận diện đa phiên bản app

Đây là ERD **sau khi** enhance được áp dụng lên base. So với base, có 4 nhóm entity mới, ứng trực tiếp với 5 yêu cầu trong đề bài:

- `MIN_SUPPORTED_APP_VERSION` (mới) — xác định rõ tập phiên bản app tối thiểu cần hỗ trợ dựa trên số liệu user thực tế, dùng làm căn cứ test hợp đồng thay vì chỉ test app mới nhất.
- `CANARY_TRAFFIC_STRATIFICATION` (mới) — đảm bảo traffic canary có đủ tỷ lệ app cũ lẫn mới, không dồn ngẫu nhiên như base.
- `CLIENT_VERSION_ERROR_METRIC` (mới) — tách error/parse-error rate theo từng phiên bản app, thay vì gộp chung.
- `SCHEMA_COMPATIBILITY_PLAN` + `ROLLBACK_EVENT` (mới) — rollback có kế hoạch đảm bảo dữ liệu ghi theo schema mới vẫn đọc được bởi code cũ.

```mermaid
erDiagram
    BACKEND_DEPLOYMENT ||--o{ MIN_SUPPORTED_APP_VERSION : "tested against"
    BACKEND_DEPLOYMENT ||--o{ CANARY_TRAFFIC_STRATIFICATION : configures
    BACKEND_DEPLOYMENT ||--o{ CLIENT_VERSION_ERROR_METRIC : measured
    BACKEND_DEPLOYMENT ||--o| SCHEMA_COMPATIBILITY_PLAN : "planned with"
    BACKEND_DEPLOYMENT ||--o{ ROLLBACK_EVENT : "may trigger on"

    APP_CLIENT_VERSION {
        string id PK
        string version
        int active_user_count
    }
    BACKEND_DEPLOYMENT {
        string id PK
        string version
        string role "stable | canary"
        int traffic_percent
    }
    MIN_SUPPORTED_APP_VERSION {
        string id PK
        string backend_version FK
        string app_version
        int active_user_count_at_test_time
        string contract_test_result
    }
    CANARY_TRAFFIC_STRATIFICATION {
        string id PK
        string backend_deployment_id FK
        string app_version
        int target_percent_of_canary
    }
    CLIENT_VERSION_ERROR_METRIC {
        string id PK
        string backend_version FK
        string app_client_version
        datetime window_start
        datetime window_end
        decimal error_rate
        decimal parse_error_rate
    }
    SCHEMA_COMPATIBILITY_PLAN {
        string id PK
        string backend_version FK
        string new_fields_written
        string backward_read_strategy "ignorable_by_old_version | requires_shim"
    }
    ROLLBACK_EVENT {
        string id PK
        string backend_deployment_id FK
        string triggered_by "client_version_error_metric | manual"
        string affected_app_versions
        string reason
        datetime triggered_at
    }
```

---

# Enhance sequence — Pre-release contract test (test đủ tập phiên bản tối thiểu)

Đây là **enhance**, cùng flow "Pre-release contract test" đã có ở base nhưng nay thay đổi theo yêu cầu thứ 1 của đề bài: xác định tập phiên bản app tối thiểu cần hỗ trợ dựa trên số liệu user thực tế, và test hợp đồng API với từng phiên bản đó.

```mermaid
sequenceDiagram
    actor Backend as Backend Engineer
    participant Analytics as User Version Analytics
    participant CI as CI/CD
    participant Test as API_CONTRACT_TEST runner

    Backend->>CI: Chuẩn bị release backend version mới
    CI->>Analytics: Lấy phân bố phiên bản app đang được user thực tế dùng
    Analytics-->>CI: Danh sách MIN_SUPPORTED_APP_VERSION (vd v3.2, v4.0, v4.5 — vẫn còn active_user_count đáng kể)
    loop Với mỗi phiên bản trong tập tối thiểu
        CI->>Test: Chạy contract test backend mới với đúng phiên bản app này
        Test-->>CI: Ghi contract_test_result cho phiên bản đó
    end
    alt Tất cả phiên bản đều pass
        CI-->>Backend: "Contract test pass cho toàn bộ tập phiên bản tối thiểu, sẵn sàng canary"
    else Có phiên bản fail (vd app cũ không parse được field mới)
        CI-->>Backend: Chặn release, yêu cầu sửa hợp đồng API hoặc thêm shim tương thích trước khi canary
    end
```

---

# Enhance sequence — Stratified canary traffic & per-version monitoring

Đây là **enhance**, cùng flow "Canary rollout" đã có ở base nhưng nay thay đổi theo yêu cầu 2, 3 và 4 của đề bài: traffic canary được stratify để bao gồm cả app cũ lẫn mới, và metric lỗi tách riêng theo `app_client_version_header` để phát hiện lỗi parse chỉ ảnh hưởng nhóm client cũ.

```mermaid
sequenceDiagram
    participant Router as Router
    participant Strat as CANARY_TRAFFIC_STRATIFICATION store
    participant Canary as Backend canary
    participant Metric as CLIENT_VERSION_ERROR_METRIC store
    actor NewApp as Client app mới nhất
    actor OldApp as Client app phiên bản cũ

    Router->>Strat: Lấy target_percent_of_canary theo từng app_version
    Note over Strat: Đảm bảo cả nhóm OldApp lẫn NewApp đều có tỷ lệ đủ lớn rơi vào canary, không chỉ dựa random đơn thuần
    NewApp->>Router: Request kèm header app_client_version=4.5
    Router->>Canary: Route theo đúng tỷ lệ stratify cho version 4.5
    OldApp->>Router: Request kèm header app_client_version=3.2
    Router->>Canary: Route theo đúng tỷ lệ stratify cho version 3.2 (đảm bảo có mặt trong canary ngay từ đầu)

    Canary->>Metric: Ghi CLIENT_VERSION_ERROR_METRIC riêng theo app_client_version_header (error_rate, parse_error_rate)

    alt parse_error_rate của nhóm OldApp (v3.2) tăng bất thường
        Metric-->>Router: Cảnh báo riêng cho nhóm v3.2, dù error_rate tổng thể vẫn thấp
        Router->>Router: Kích hoạt ROLLBACK_EVENT(affected_app_versions=v3.2)
        Note over Metric: Nhờ tách theo version, lỗi chỉ ảnh hưởng thiểu số client cũ không bị chìm trong tỷ lệ lỗi tổng thể
    else Mọi nhóm version đều ổn định
        Metric-->>Router: Cho phép tiếp tục tăng traffic canary
    end
```

---

# Enhance sequence — Backward-compatible rollback

Đây là **enhance**, cùng flow "Rollback" đã có ở base nhưng nay thay đổi hoàn toàn theo yêu cầu thứ 5 của đề bài: nếu canary đã ghi dữ liệu theo schema mới, rollback phải đảm bảo phiên bản cũ (đang phục vụ client legacy) vẫn đọc được dữ liệu đó, không chỉ đơn giản revert code.

```mermaid
sequenceDiagram
    participant Router as Router
    participant Canary as Backend canary
    participant Stable as Backend stable (bản cũ)
    participant Plan as SCHEMA_COMPATIBILITY_PLAN store
    participant DB as Shared Database

    Note over Router: ROLLBACK_EVENT vừa được kích hoạt (từ flow "Stratified canary monitoring")
    Router->>Plan: Tra SCHEMA_COMPATIBILITY_PLAN của backend_version canary
    Plan-->>Router: backward_read_strategy đã định nghĩa trước khi canary bắt đầu

    alt backward_read_strategy = ignorable_by_old_version
        Note over DB: Field mới canary đã ghi (vd trường mở rộng cho tính năng mới) được thiết kế để code cũ có thể bỏ qua an toàn, không gây lỗi parse
        Router->>Stable: Chuyển toàn bộ traffic về Stable
        Stable->>DB: Đọc dữ liệu bình thường, bỏ qua field mới không quen thuộc
        Stable-->>Router: Phục vụ đúng cho cả client cũ lẫn mới
    else backward_read_strategy = requires_shim
        Router->>Stable: Chuyển traffic về Stable, nhưng kèm 1 shim đọc dữ liệu
        Stable->>DB: Đọc dữ liệu (có thể ở schema mới)
        DB-->>Stable: Trả dữ liệu thô
        Stable->>Stable: Shim chuyển đổi dữ liệu về đúng hình dạng mà code cũ hiểu được
        Stable-->>Router: Phục vụ đúng, không gãy tương thích ngược lần nữa
    end
    Note over Plan: Vì SCHEMA_COMPATIBILITY_PLAN đã được chuẩn bị từ trước khi canary bắt đầu ghi dữ liệu mới, rollback không bị động lúc sự cố xảy ra
```
