# Sequence Diagram — Enhance: Autocomplete Address

Đây là **enhance**, flow hoàn toàn mới: autocomplete xử lý địa chỉ viết tắt/thiếu dấu/thứ tự phường-quận-thành phố khác nhau, gợi ý đúng địa chỉ chuẩn hóa mà không cần người dùng gõ chính xác định dạng đầy đủ.

```mermaid
sequenceDiagram
    actor Buyer
    participant WebApp as Real Estate App
    participant Autocomplete as Address Autocomplete Service
    participant Index as Address Index

    Buyer->>WebApp: Gõ địa chỉ không thống nhất (viết tắt, thiếu dấu, sai thứ tự)
    WebApp->>Autocomplete: Autocomplete request (partial input)

    Autocomplete->>Autocomplete: Chuẩn hóa input (bỏ dấu, mở rộng viết tắt, không phụ thuộc thứ tự token)
    Autocomplete->>Index: Query trên alias_text/normalized_address đã chuẩn hóa tương tự
    Index-->>Autocomplete: Danh sách địa chỉ chuẩn hóa khớp, kèm tọa độ

    Autocomplete-->>WebApp: Gợi ý địa chỉ chuẩn hóa
    WebApp-->>Buyer: Hiển thị gợi ý, buyer chọn 1 địa chỉ đúng ý
```
