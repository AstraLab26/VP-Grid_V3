# VP-Grid V3 (Sub engine AA/BB only)

## Giới thiệu
`VP-Grid_V3.mq5` là phiên bản dựa trên `VP-Grid.mq5` và bổ sung **chế độ lệnh phụ (sub)** chạy độc lập bằng **magic riêng**:
- Lệnh chính: dùng các magic mặc định `MagicAA/MagicBB/...` theo đúng logic hiện có.
- Lệnh phụ: chỉ chạy **2 loại**: `AA phụ` và `BB phụ`.
- **Main reset không đóng/khởi động lại lệnh phụ** vì các thao tác đóng/clear của main chỉ lọc theo magic main.

## Sub engine kích hoạt khi nào
Sub sẽ được kích hoạt khi:
- `EnableSub = true`
- Và khoảng cách giá hiện tại so với **base của lệnh chính** đủ lớn:
  - `|Ask/Bid(current) - baseMain| >= SubDistanceFromMainBasePips` (quy đổi theo pip)

Sau khi kích hoạt:
- `baseSub = giá hiện tại`
- Sub chỉ đặt virtual pendings cho:
  - `AA phụ` (magic riêng)
  - `BB phụ` (magic riêng)

## Input mới của V3
Trong nhóm **14. SUB MODE (AA/BB only)**:
- `EnableSub`: bật/tắt chế độ sub.
- `SubDistanceFromMainBasePips`: X pip (sub chỉ bắt đầu khi cách base main >= X pip).
- `MagicSubOffset`: offset magic phụ:
  - `MagicAA_Sub = MagicNumber + MagicSubOffset`
  - `MagicBB_Sub = MagicNumber + MagicSubOffset + 1`

## Reset độc lập cho sub (Reset mode 12)
V3 hiện triển khai reset độc lập cho sub trong:
- **RESET WHEN LEVELS MATCH (mode 12)**:
  - Điều kiện reset sub dùng:
    - khoảng cách từ baseSub (>= `LevelMatchMinDistancePips`)
    - và ngưỡng `LevelMatchSessionTargetUSD`
  - Nếu bật `EnableLockProfit`, phần “tiết kiệm %” từ TP sẽ được trừ vào P/L dùng xét reset sub qua biến `sessionLockedProfitSub`.

## Các chế độ khác của sub
Hiện tại trong V3, sub **đang không có bản tách riêng đầy đủ** cho tất cả cơ chế reset/stop/trailing giống main.
Cụ thể:
- `ManageGridOrdersSub()` vẫn tuân thủ các điều kiện start/filter chung (trading hours/weekday/ADX/RSI) theo cài đặt của EA.
- Sub chưa có cơ chế trailing/stop riêng (ngoài reset mode 12) theo đúng nghĩa “tách 100% toàn bộ state” cho mọi loại reset.

## Ghi chú kỹ thuật quan trọng
- Virtual pendings của sub được lưu thêm `basePriceAtAdd`, để khi xử lý “điểm chạm base” không bị lệch do main/sub có base khác nhau.
- Khi main reset/clear:
  - đóng positions và clear virtual pendings chỉ theo magic main
  - không chạm magic phụ (`MagicAA_Sub`, `MagicBB_Sub`)

## Khuyến nghị test
1. Chạy thử với `EnableSub=true` và `SubDistanceFromMainBasePips` lớn để đảm bảo sub chỉ vào sau khi main đã start/reset.
2. Kiểm tra log/Journal cho các dòng:
   - `Sub activated: ...`
   - `Sub Level match reset ...` (nếu bật reset mode 12)

