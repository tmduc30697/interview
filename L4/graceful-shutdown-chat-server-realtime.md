# Chat server real-time cho ứng dụng nhắn tin

**Hệ thống:** Một backend chat real-time giữ kết nối WebSocket cho hàng chục nghìn user, mỗi kết nối gắn với một session hội thoại.

**Vai trò của flow:** Đảm bảo khi một chat server instance cần tắt (maintenance/deploy), user không bị mất tin nhắn đang gửi và được chuyển êm sang instance khác.

**Yêu cầu cụ thể:**
- Tin nhắn đang trong hàng đợi gửi (chưa được ACK bởi client) tại thời điểm shutdown phải được đảm bảo không mất — chuyển giao lại cho hàng đợi bền (message queue/DB) để instance khác tiếp tục gửi, không được chỉ giữ trong memory rồi mất khi tắt.
- Khi ngắt kết nối WebSocket để drain, phải gửi kèm mã lý do (close code) riêng cho "server draining" khác với "lỗi bất thường", để client phân biệt và tự reconnect ngay, không hiển thị lỗi gây hoang mang cho người dùng.
- Đảm bảo tính idempotent: nếu một tin nhắn đã được gửi thành công tới client trước khi server bị ngắt giữa lúc gửi ACK, khi reconnect vào instance mới, tin nhắn không được gửi lại trùng lặp.
- Xử lý trường hợp user đang gõ (typing indicator) hoặc đang trong cuộc gọi voice/video khi server drain — trạng thái này phải được chuyển giao hoặc kết thúc rõ ràng, không để lại trạng thái "đang gõ" treo vĩnh viễn ở phía người nhận.
- Thiết kế cơ chế rolling restart cho toàn cluster chat server sao cho tại mọi thời điểm, tổng số kết nối bị buộc reconnect đồng thời không vượt quá X% tổng số user online, tránh gây spike tải lên hệ thống presence/auth khi hàng loạt client reconnect cùng lúc.

---

# Base ERD — Chat server real-time trước khi có graceful shutdown

Đây là **base**: mô hình dữ liệu suy luận từ ngữ cảnh đề bài, thể hiện trạng thái hệ thống **trước khi** chat server biết xử lý shutdown an toàn. Đề bài nói tới backend giữ kết nối WebSocket cho hàng chục nghìn user, mỗi kết nối gắn với 1 session hội thoại — nên base chỉ cần đủ: user, cuộc hội thoại, tin nhắn, và kết nối WebSocket hiện tại của user. Tin nhắn chưa gửi/chưa ACK chỉ tồn tại trong bộ nhớ của instance đang giữ kết nối, không có nơi lưu bền vững nào khác — đây chính là lỗ hổng mà enhance sẽ vá. Chưa có trạng thái draining, chưa có close code phân biệt, chưa có theo dõi typing/call, chưa có kế hoạch giới hạn % reconnect đồng thời — tất cả là phần enhance.

```mermaid
erDiagram
    USER ||--o{ CONNECTION : "opens"
    USER ||--o{ MESSAGE : sends
    CONVERSATION ||--o{ MESSAGE : contains

    USER {
        string id PK
        string name
    }
    CONVERSATION {
        string id PK
    }
    MESSAGE {
        string id PK
        string conversation_id FK
        string sender_id FK
        string content
        string status "pending | sent"
        datetime created_at
    }
    CONNECTION {
        string id PK
        string user_id FK
        string server_instance_id
        datetime connected_at
    }
```

Ghi chú: `MESSAGE.status=pending` (đã tạo nhưng chưa gửi được qua WebSocket, hoặc đã gửi nhưng chưa nhận ACK từ client) chỉ được theo dõi trong bộ nhớ tiến trình của `server_instance_id` đang giữ `CONNECTION`, không có bảng/queue bền vững nào lưu lại phần này ở base.

---

# Base sequence — Send message (chỉ giữ trong memory, mất khi instance tắt)

Đây là **base**, flow "Gửi tin nhắn qua WebSocket" ở trạng thái hiện tại — tin nhắn đang chờ gửi/chờ ACK chỉ tồn tại trong bộ nhớ của instance đang giữ kết nối, và khi instance tắt (không phân biệt lý do), kết nối bị đóng đột ngột không kèm mã lý do rõ ràng. Flow này liên quan mật thiết tới enhance vì toàn bộ yêu cầu của đề bài đều nhằm vá đúng những lỗ hổng này.

```mermaid
sequenceDiagram
    actor Sender
    participant I1 as Chat Server Instance A
    actor Receiver

    Sender->>I1: Gửi tin nhắn M1 (qua WebSocket)
    I1->>I1: Lưu M1 vào hàng đợi gửi trong memory (status=pending)
    I1->>Receiver: Đẩy M1 qua WebSocket
    Note over I1: Đang chờ Receiver gửi ACK xác nhận đã nhận M1

    Note over I1: Vận hành trigger deploy/maintenance, tắt Instance A ngay bây giờ, không phân biệt shutdown chủ động hay crash
    I1--xI1: Process tắt, hàng đợi gửi trong memory (bao gồm M1 nếu chưa kịp nhận ACK) biến mất hoàn toàn

    Receiver->>Receiver: Kết nối WebSocket bị đóng đột ngột không kèm mã lý do, client hiển thị lỗi kết nối chung chung
    Note over Receiver: Người dùng không biết đây là do server bảo trì hay lỗi thật, tự đoán và tự reconnect sau vài giây với backoff ngẫu nhiên
```

---

# Enhance ERD — Thêm hàng đợi bền vững, presence có TTL, kế hoạch giới hạn reconnect

Đây là **enhance**, mô hình dữ liệu sau khi áp toàn bộ đề bài lên base. So với base, các thay đổi:
- `MESSAGE` có thêm trạng thái `queued_durable` và cột `delivered_at`/`acked_by_client_at` — tin nhắn chưa được client ACK nay được ghi vào store bền vững thay vì chỉ giữ trong memory, và server theo dõi được đã gửi/đã ACK để tránh gửi trùng khi reconnect — đáp ứng yêu cầu 1 và yêu cầu 3.
- `CONNECTION` có thêm trạng thái `draining` và cột `close_code` — đáp ứng yêu cầu 2 (mã lý do riêng cho "server draining").
- Entity mới `PRESENCE_STATE` — đại diện trạng thái tạm thời như đang gõ hoặc đang trong cuộc gọi, có `expires_at` để không bao giờ treo vĩnh viễn — đáp ứng yêu cầu 4.
- Entity mới `CLUSTER_DRAIN_PLAN` — giới hạn tổng số kết nối bị buộc reconnect đồng thời không vượt quá X% tổng số user online — đáp ứng yêu cầu 5.

```mermaid
erDiagram
    USER ||--o{ CONNECTION : "opens"
    USER ||--o{ MESSAGE : sends
    CONVERSATION ||--o{ MESSAGE : contains
    CONVERSATION ||--o{ PRESENCE_STATE : "has"
    CLUSTER_DRAIN_PLAN ||--o{ CONNECTION : coordinates

    USER {
        string id PK
        string name
    }
    CONVERSATION {
        string id PK
    }
    MESSAGE {
        string id PK
        string conversation_id FK
        string sender_id FK
        string content
        string status "pending | queued_durable | delivered | acked"
        datetime delivered_at
        datetime acked_by_client_at
    }
    CONNECTION {
        string id PK
        string user_id FK
        string server_instance_id
        string status "connected | draining | closed"
        int close_code "vd 4000=draining, khac voi 1006=loi bat thuong"
        datetime connected_at
    }
    PRESENCE_STATE {
        string id PK
        string conversation_id FK
        string user_id FK
        string type "typing | in_call"
        datetime started_at
        datetime expires_at
    }
    CLUSTER_DRAIN_PLAN {
        string id PK
        string deploy_id
        int total_online_users
        float max_percent_reconnect_simultaneous
        int current_batch_reconnecting
    }
```

---

# Enhance sequence — Send message (chuyển giao hàng đợi bền vững, không gửi trùng khi reconnect)

Đây là **enhance** của flow `send-message` đã có ở base, đáp ứng **yêu cầu 1 và yêu cầu 3** của đề bài. So với base: tin nhắn chưa được ACK tại thời điểm shutdown được ghi vào store bền vững (`status=queued_durable`) thay vì chỉ giữ trong memory, để instance khác tiếp tục gửi. Song song đó, mỗi tin nhắn có `delivered_at` được ghi lại ngay khi gửi thành công, nên nếu server bị ngắt giữa lúc gửi ACK, instance mới khi reconnect biết tin nhắn đã delivered và không gửi lại trùng.

```mermaid
sequenceDiagram
    actor Sender
    participant I1 as Chat Server Instance A (đang drain)
    participant Store as Durable Message Store
    actor Receiver
    participant I2 as Chat Server Instance B

    Sender->>I1: Gửi tin nhắn M1
    I1->>Store: INSERT MESSAGE(status=pending)
    I1->>Receiver: Đẩy M1 qua WebSocket
    I1->>Store: UPDATE MESSAGE SET status=delivered, delivered_at=now (ghi ngay khi gửi thành công, trước khi có ACK)

    Note over I1: Instance A nhận tín hiệu shutdown ngay lúc này, chưa kịp nhận ACK từ Receiver cho M1
    I1->>Store: Đảm bảo mọi MESSAGE chưa có acked_by_client_at đều đã ở status=delivered hoặc queued_durable (không còn nằm riêng trong memory)
    I1->>I1: Đóng kết nối tới Receiver (xem chi tiết ở flow drain-disconnect-close-code)

    Receiver->>I2: Reconnect vào Instance B
    I2->>Store: SELECT MESSAGE WHERE conversation_id=... AND acked_by_client_at IS NULL
    Store-->>I2: M1 (status=delivered, delivered_at đã có, acked_by_client_at=null)
    I2->>I2: Vì M1 đã delivered_at trước đó, không gửi lại M1, chỉ chờ ACK
    Receiver->>I2: Gửi ACK cho M1 (client vốn đã nhận M1 từ trước khi mất kết nối)
    I2->>Store: UPDATE MESSAGE SET status=acked, acked_by_client_at=now
```

---

# Enhance sequence — Ngắt kết nối để drain kèm close code riêng biệt

Đây là flow **enhance** hoàn toàn mới so với base, đáp ứng **yêu cầu 2** của đề bài: khi ngắt kết nối WebSocket để drain, server gửi kèm close code riêng cho "server draining" (ví dụ 4000), khác với close code lỗi bất thường (ví dụ 1006), để client phân biệt và tự reconnect ngay thay vì hiển thị lỗi gây hoang mang.

```mermaid
sequenceDiagram
    actor Client
    participant I1 as Chat Server Instance A (đang drain)
    participant I2 as Chat Server Instance B

    Note over I1: Instance A nhận tín hiệu shutdown, chuyển CONNECTION.status=draining
    I1->>Client: WebSocket close(code=4000, reason="server_draining")
    Client->>Client: Nhận diện code=4000 khác với code=1006 (lỗi bất thường)

    alt code=4000 server_draining
        Client->>Client: Không hiển thị lỗi cho người dùng, tự động reconnect ngay lập tức
        Client->>I2: Mở kết nối WebSocket mới
        I2-->>Client: Kết nối thành công, tiếp tục phiên chat bình thường
    else code=1006 hoặc các mã lỗi bất thường khác
        Client->>Client: Hiển thị trạng thái "mất kết nối", reconnect theo backoff tăng dần
    end
```

---

# Enhance sequence — Chuyển giao/kết thúc rõ ràng typing indicator và cuộc gọi khi drain

Đây là flow **enhance** hoàn toàn mới so với base, đáp ứng **yêu cầu 4** của đề bài: khi server drain lúc user đang gõ hoặc đang trong cuộc gọi voice/video, các `PRESENCE_STATE` này phải được chuyển giao hoặc kết thúc rõ ràng, không để lại trạng thái "đang gõ" treo vĩnh viễn ở phía người nhận.

```mermaid
sequenceDiagram
    actor UserA as User A (đang gõ)
    participant I1 as Chat Server Instance A (đang drain)
    participant Store as PRESENCE_STATE store
    actor UserB as User B (đang xem "A đang gõ...")
    participant I2 as Chat Server Instance B

    UserA->>I1: Bắt đầu gõ trong conversation C
    I1->>Store: INSERT PRESENCE_STATE(type=typing, expires_at=now+10s)
    I1->>UserB: Đẩy sự kiện "A đang gõ..."

    Note over I1: Instance A nhận tín hiệu shutdown giữa lúc User A vẫn đang gõ
    I1->>Store: UPDATE PRESENCE_STATE SET expires_at=now (hết hạn ngay, không chờ tới +10s)
    I1->>UserB: Đẩy sự kiện "A đã ngừng gõ" (kết thúc rõ ràng thay vì treo)
    I1->>UserA: Đóng kết nối (close_code=4000 server_draining)
    UserA->>I2: Reconnect, tiếp tục gõ nếu vẫn đang gõ
    I2->>Store: INSERT PRESENCE_STATE mới (type=typing) nếu UserA còn đang gõ

    Note over I1,Store: Với cuộc gọi voice/video, tương tự nhưng KHÔNG tự kết thúc cuộc gọi
    Note over I1,Store: PRESENCE_STATE(type=in_call) được đánh dấu status=transferring, tín hiệu signaling (SDP/ICE) được chuyển tiếp sang Instance B để cuộc gọi tiếp tục không gián đoạn, chỉ kết thúc hẳn nếu instance mới không nhận chuyển giao kịp trong thời gian giới hạn
```

---

# Enhance sequence — Rolling restart toàn cluster giới hạn % kết nối reconnect đồng thời

Đây là flow **enhance** hoàn toàn mới so với base, đáp ứng **yêu cầu 5** của đề bài: rolling restart toàn cluster chat server sao cho tại mọi thời điểm, tổng số kết nối bị buộc reconnect đồng thời không vượt quá X% tổng số user online, tránh spike tải lên hệ thống presence/auth.

```mermaid
sequenceDiagram
    participant Orchestrator as Deploy Orchestrator
    participant Plan as CLUSTER_DRAIN_PLAN
    participant I1 as Instance 1 (5000 kết nối)
    participant I2 as Instance 2 (5000 kết nối)
    participant I3 as Instance 3 (5000 kết nối)
    participant Auth as Presence/Auth Service

    Orchestrator->>Plan: Bắt đầu rolling restart cluster (total_online_users=15000, max_percent_reconnect_simultaneous=10%)
    Note over Plan: Ngân sách reconnect đồng thời tối đa = 1500 kết nối tại 1 thời điểm

    Orchestrator->>Plan: Xin phép drain Instance 1 (5000 kết nối)
    Plan-->>Orchestrator: Từ chối drain toàn bộ cùng lúc, 5000 > ngân sách 1500

    Orchestrator->>Plan: Xin phép drain theo lô 1500 kết nối của Instance 1
    Plan-->>Orchestrator: OK, current_batch_reconnecting=1500
    Orchestrator->>I1: Drain lô 1 (1500 kết nối), gửi close_code=4000 cho từng kết nối trong lô
    I1->>Auth: 1500 client reconnect, xác thực lại
    Auth-->>I1: Xử lý xong, tải trong ngưỡng cho phép
    I1->>Plan: Lô 1 hoàn tất, current_batch_reconnecting=0

    Orchestrator->>Plan: Xin phép drain lô 2 (1500 kết nối tiếp theo của Instance 1)
    Plan-->>Orchestrator: OK
    Orchestrator->>I1: Drain lô 2
    I1->>Auth: 1500 client reconnect

    Note over Orchestrator,Plan: Lặp lại theo từng lô 1500 kết nối cho tới khi Instance 1 drain xong toàn bộ, rồi mới chuyển sang Instance 2 và Instance 3 theo cùng cơ chế, đảm bảo Auth/presence không bao giờ nhận quá 1500 lượt reconnect cùng lúc
```
