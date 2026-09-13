# Sequence - Base - Flow "change-variant-selection"

Đây là **base**: khi khách đổi biến thể (ví dụ từ size M sang size L) trong lúc đang giữ reservation, hệ thống nhả reservation cũ rồi giữ reservation mới bằng 2 bước tách rời. Flow này được chọn vì đây là cách làm mà yêu cầu 2 chỉ ra là có vấn đề: khoảng hở giữa 2 bước có thể khiến biến thể mới bị người khác giữ mất, hoặc biến thể cũ bị giữ dư thừa nếu bước sau thất bại.

```mermaid
sequenceDiagram
    actor Customer
    participant App as Sàn thời trang
    participant VariantM as Variant (size M)
    participant VariantL as Variant (size L)

    Note over VariantM: Customer đang giữ reservation active cho size M

    Customer->>App: đổi lựa chọn từ size M sang size L
    App->>VariantM: bước 1, nhả reservation cũ, quantity += 1

    Note over VariantL: Khoảng hở giữa 2 bước, người khác có thể giữ mất size L ngay lúc này

    App->>VariantL: bước 2, giữ reservation mới cho size L

    alt Không ai giành size L trong khoảng hở
        VariantL-->>App: giữ thành công
        App-->>Customer: đổi biến thể thành công
    else Size L đã bị người khác giữ mất trong khoảng hở
        VariantL-->>App: hết hàng, không giữ được
        App-->>Customer: báo lỗi, đồng thời size M cũ cũng đã bị nhả oan nếu khách muốn quay lại
    end
```
