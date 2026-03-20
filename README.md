# VP-Grid V3 (Main/Sub chạy riêng biệt)

## 1) Tổng quan
`VP-Grid_V3.mq5` là bản nâng cấp từ `VP-Grid.mq5`, trong đó:
- **Main engine** chạy đầy đủ AA/BB/CC/DD như bản gốc.
- **Sub engine** chạy riêng với **AA/BB** bằng magic riêng.
- Mục tiêu thiết kế: **giống logic giữa Main và Sub, nhưng state tách riêng** (reset, stop/start, trailing, session, lock, balance, thông báo).

---

## 2) Input Sub mode (nhóm 14)
- `EnableSub`: bật/tắt Sub engine.
- `SubDistanceFromMainBasePips`: Sub chỉ được kích hoạt khi `|price - baseMain| >= X pips`.
- `MagicSubOffset`:
  - `MagicAA_Sub = MagicNumber + MagicSubOffset`
  - `MagicBB_Sub = MagicNumber + MagicSubOffset + 1`

---

## 3) Kích hoạt Sub
Sub được kích hoạt khi:
- `EnableSub = true`
- Sub chưa active
- Không bị chặn bởi các cờ stop của Sub (`eaStoppedByTargetSub`, `eaStoppedByScheduleSub`, `eaStoppedByWeekdaySub`, `eaStoppedByAdxSub`, `eaStoppedByRsiSub`, `eaStoppedByRestartDelaySub`)
- Đủ điều kiện khoảng cách với `baseMain` theo `SubDistanceFromMainBasePips`

Khi kích hoạt:
- `baseSub = giá hiện tại`
- tạo `gridLevelsSub`
- reset state phiên của Sub
- chỉ quản lý lệnh `AA_Sub` và `BB_Sub`

---

## 4) Những phần đã tách riêng Main/Sub

### 4.1 Session / PnL / Lock
- `sessionStartTime`, `sessionStartBalance` tách riêng cho Main/Sub.
- `sessionClosedProfit`, `sessionClosedProfitSub` tách riêng.
- `sessionLockedProfit`, `sessionLockedProfitSub` tách riêng.
- **Locked reserve** tách riêng:
  - `lockedProfitReserveMain`
  - `lockedProfitReserveSub`

### 4.2 Scaling theo vốn
- Main dùng `sessionMultiplier`
- Sub dùng `sessionMultiplierSub`
- Lot đặt lệnh Sub dùng hàm lot riêng (`GetLotForLevelSub`, `GetLotForLevelBBSub`).

### 4.3 Trailing (gồng lãi tổng)
- Main: `gongLaiMode`, `DoGongLaiTrailing()`
- Sub: `gongLaiModeSub`, `DoGongLaiTrailingSub()`
- Sub có logic riêng cho:
  - vào trailing
  - drop mode return/lock
  - reset khi SL trailing bị hit

### 4.4 Reset mode
- Main giữ các mode reset như bản gốc.
- Sub có reset riêng:
  - Session profit target (mode 11 theo logic target phiên)
  - Level match / distance + session target (mode 12)
  - Trailing lock / trailing SL hit
- Tất cả reset Sub đi qua `DisableSubEngineAndMaybeRestart()`.

### 4.5 Stop/Start filter state machine
Main và Sub có cờ riêng cho:
- Daily stop
- Trading hours
- Weekday schedule
- ADX start filter
- RSI start filter
- Restart delay

Các hàm Sub riêng:
- `CheckDailyRolloverAndAutoRestartSub()`
- `CheckTradingHoursAndAutoRestartSub()`
- `CheckWeekdayAndAutoRestartSub()`
- `CheckADXStartAndAutoRestartSub()`
- `CheckRSIStartAndAutoRestartSub()`
- `DailyStopOnResetAccumulateAndMaybeStopSub()`
- `TradingHoursStopOnResetIfNeededSub()`
- `WeekdayStopOnResetIfNeededSub()`

### 4.6 Balance
- Main giữ `DoBalanceAll()` như cũ.
- Sub có balance riêng cho chế độ open/noTP across base:
  - `BalanceOpenAcrossBaseNoTP_Sub()`
  - dùng pool/cooldown riêng của Sub (`sessionClosedProfitRemainingSub`, `lastBalance...Sub`)

> Luu y: Sub chỉ có AA/BB, nên không có CC/DD trong balance của Sub.

### 4.7 Virtual pending / re-arm
- Main clear/đóng theo magic Main.
- Sub clear/đóng theo magic Sub.
- `VirtualPendingEntry.basePriceAtAdd` giúp pending của mỗi engine bám đúng base đã tạo.

### 4.8 Vẽ chart
- Base Main: đường chấm **vàng**
- Base Sub: đường chấm **trắng**
- Vùng mở lệnh:
  - Main: vùng mờ **trắng**
  - Sub: vùng mờ **vàng**

---

## 5) Notification (Main/Sub tách riêng)

### 5.1 Reason tách rõ
- Main: reason bình thường (`Session profit target reached - reset`, ...)
- Sub: reason có tiền tố `Sub ...` (ví dụ `Sub Level match reset: waiting 5 min before restart`)

### 5.2 Tránh lỗi lặp "Sub Sub ..."
- Hàm `ScheduleRestartDelayAfterResetSub()` đã xử lý tránh cộng tiền tố `Sub` 2 lần.

### 5.3 Nội dung status trong thông báo
`SendResetNotification()` sẽ tự nhận biết reason là Sub hay Main để hiển thị đúng state:
- Daily progress: `dailyResetProfit` hoặc `dailyResetProfitSub`
- Trading/weekday stop flags: bản Main hoặc Sub tương ứng
- ADX/RSI stop flags: bản Main hoặc Sub tương ứng
- Restart delay: `restartDelayUntil` hoặc `restartDelayUntilSub`

### 5.4 Ví dụ reason
- Main: `Session profit target reached - reset`
- Sub: `Sub Level match reset: waiting 5 min before restart`

---

## 6) Luồng reset Sub sau khi kích hoạt
Khi Sub reset:
1. Đóng toàn bộ vị thế + pending của Sub.
2. Cập nhật scaling Sub cho phiên sau.
3. Áp dụng daily/trading-hours/weekday stop của Sub.
4. Áp dụng restart delay của Sub (nếu bật).
5. Nếu không bị stop, kiểm tra ADX/RSI start filter của Sub.
6. Chờ điều kiện khoảng cách tới `baseMain` để kích hoạt lại Sub.

---

## 7) Lưu ý vận hành
- Main reset **không đóng lệnh Sub**.
- Sub reset **không đóng lệnh Main**.
- Sub luôn phụ thuộc điều kiện khoảng cách so với `baseMain` để vào lại.
- Nếu dùng filter thời gian/ngày/chỉ báo, Main và Sub giữ cờ chặn riêng.

---

## 8) Checklist test nhanh
1. Bật `EnableSub=true`, đặt `SubDistanceFromMainBasePips` dễ chạm.
2. Xác nhận log có `Sub activated: ...`.
3. Kích hoạt một reset của Main -> xác nhận Sub không bị đóng.
4. Kích hoạt một reset của Sub -> xác nhận Main vẫn chạy.
5. Kiểm tra notification:
   - Main reason không có `Sub`
   - Sub reason có tiền tố `Sub ...`
6. Kiểm tra restart delay:
   - Main delay hiển thị theo Main
   - Sub delay hiển thị theo Sub

