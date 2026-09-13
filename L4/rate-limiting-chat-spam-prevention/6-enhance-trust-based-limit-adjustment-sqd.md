# Enhance sequence — Rate limit động theo uy tín tài khoản

Đây là **enhance**, flow mới phát sinh từ enhance — chọn `RATE_LIMIT_POLICY` áp dụng cho token bucket của mỗi user dựa trên `trust_level`, đảm bảo tài khoản mới/chưa xác thực bị giới hạn chặt hơn trong khi tài khoản lâu năm có lịch sử hành vi tốt không bị chặn quá nghiêm. Đúng yêu cầu cuối của đề bài.

```mermaid
sequenceDiagram
    actor NewUser as User mới tạo tài khoản
    actor OldUser as User lâu năm, hành vi tốt
    participant App as Chat Service
    participant Policy as RATE_LIMIT_POLICY store
    participant Bucket as RATE_LIMIT_BUCKET store
    participant Trust as Trust Level Evaluator (background)

    par Đánh giá trust_level định kỳ cho mọi user
        Trust->>Trust: Tính trust_level dựa trên tuổi tài khoản, is_verified, lịch sử bị flag/report
        Trust->>App: UPDATE USER SET trust_level=new (tài khoản mới, chưa xác thực)
        Trust->>App: UPDATE USER SET trust_level=trusted (tài khoản lâu năm, không có report xấu)
    end

    NewUser->>App: Gửi tin nhắn trong group
    App->>Policy: Tra RATE_LIMIT_POLICY(trust_level=new, conversation_type=group)
    Policy-->>App: burst_capacity=2, sustained_rate=0.5 tin/giây (chặt)
    App->>Bucket: Áp dụng policy chặt lên RATE_LIMIT_BUCKET của NewUser
    App-->>NewUser: Gửi thành công, nhưng burst tối đa chỉ 2 tin liên tiếp

    OldUser->>App: Gửi tin nhắn trong group
    App->>Policy: Tra RATE_LIMIT_POLICY(trust_level=trusted, conversation_type=group)
    Policy-->>App: burst_capacity=8, sustained_rate=3 tin/giây (khoan dung)
    App->>Bucket: Áp dụng policy khoan dung lên RATE_LIMIT_BUCKET của OldUser
    App-->>OldUser: Gửi thành công, burst tới 8 tin liên tiếp không bị chặn

    Note over Policy,Bucket: Cùng 1 hành vi gõ dồn dập nhưng tài khoản mới bị giới hạn chặt hơn để kiểm soát rủi ro spam, tài khoản lâu năm được khoan dung hơn nhờ lịch sử hành vi tốt
```
