# TÀI LIỆU KỸ THUẬT VÀ HƯỚNG DẪN VẬN HÀNH SNIPER EA V4 PRODUCTION

Hệ thống Giao dịch Thuật toán Tự động Đa tầng dành cho XAUUSD (Vàng) và Ngoại hối (Forex) trên nền tảng MetaTrader 5 (MT5).

Tác giả: Nguyễn Quang Tú (qtusdev) - System Architect & Trading Quant.


---

## I. TỔNG QUAN KIẾN TRÚC HỆ THỐNG (SYSTEM ARCHITECTURE)

Sniper EA V4 Production được xây dựng dựa trên kiến trúc động cơ kép độc lập (Dual-Engine Architecture), cho phép người dùng tùy biến triệt để giữa phong cách lướt sóng nhanh (Scalping) và phong cách nắm giữ xu hướng theo dòng tiền tổ chức (Institutional Swing Hold).

```
                            SNIPER EA V4 PRODUCTION
                                      │
           ┌──────────────────────────┴──────────────────────────┐
           ▼                                                     ▼
[ENGINE 1: EMA M1/M5 SCALP]                      [ENGINE 2: INSTITUTIONAL SWING HOLD]
- Khung: M1 kết hợp M5                           - Khung: H1 -> M15 -> M5 -> M1 (4 Tầng)
- Logic: Đồng thuận EMA9/EMA21                   - Logic: 100% Pure Price Action (Zero Indicator)
- Dynamic Trend Ribbon                           - Thanh khoản: Liquidity Sweep (EQH/EQL)
- Quản trị: 6 mốc TP R:R + ATR Trailing          - Xung lực: Displacement + True Structure BOS
- Bảo hiểm: Time-Stop & Reverse Close            - Điểm vào: FVG 50% CE Retest + Micro MSS (M1)
                                                 - Quản trị: TP1 (Internal) -> TP2 (External) -> Runner
                                                 - Trailing: Post-Entry Structure Trailing
```

---

## II. CHI TIẾT CƠ CHẾ HOẠT ĐỘNG CỦA ENGINE ĐỊNH CHẾ (INSTITUTIONAL HOLD)

Engine Institutional Swing Hold được thiết kế theo quy trình 6 bước chuẩn định lượng, hoạt động độc lập 100% không sử dụng bất kỳ chỉ báo kỹ thuật nào (kể cả ATR khi bật chế độ Pure Price Action):

### 1. Tầng 1: Khảo sát Cấu trúc Xu hướng Lớn (H1 Structure Bias)
* **Khung thời gian:** `InpHTFBiasTF` (Mặc định: PERIOD_H1).
* **Cơ chế:** Thu thập các đỉnh/đáy cấu trúc (Swing Highs và Swing Lows) đã được xác nhận với độ mạnh `InpPivotStrength`.
* **Điều kiện Bullish Bias:** Đỉnh sau cao hơn đỉnh trước (`HH`) VÀ Đáy sau cao hơn đáy trước (`HL`) có thứ tự thời gian xen kẽ hợp lệ, hoặc nến H1 đóng cửa phá vỡ đỉnh swing gần nhất (`HTF BOS`) kết hợp đáy nâng cao.
* **Điều kiện Bearish Bias:** Đỉnh sau thấp hơn đỉnh trước (`LH`) VÀ Đáy sau thấp hơn đáy trước (`LL`) có thứ tự thời gian xen kẽ hợp lệ, hoặc nến H1 đóng cửa đâm thủng đáy swing gần nhất (`HTF BOS`) kết hợp đỉnh hạ thấp.
* **Bộ lọc Chặn Ngược Chiều (`InpFilterAgainstHTFBias`):** Tuyệt đối không mở lệnh BUY khi H1 là Bearish; tuyệt đối không mở lệnh SELL khi H1 là Bullish.

### 2. Tầng 2: Xác định Vị trí và Bể Thanh Khoản Ngoại vi (M15 Location & External Target)
* **Khung thời gian:** `InpHTFLocationTF` (Mặc định: PERIOD_M15).
* **Cơ chế:** Quét biên độ dao động `InpRangeLookbackBars` để tính:
  $$\text{Equilibrium} = \frac{\text{HTF High} + \text{HTF Low}}{2}$$
* **Location Gate:**
  * Lệnh BUY chỉ được kích hoạt khi giá đang nằm trong vùng **Discount Zone** (dưới Equilibrium), trừ khi H1 đang trong xu hướng tăng mạnh tiếp diễn.
  * Lệnh SELL chỉ được kích hoạt khi giá đang nằm trong vùng **Premium Zone** (trên Equilibrium), trừ khi H1 đang trong xu hướng giảm mạnh tiếp diễn.
* **External Target:** Đỉnh HTF High (cho lệnh BUY) hoặc Đáy HTF Low (cho lệnh SELL).

### 3. Tầng 2.5: Quét Thanh Khoản và Nến Bung Xung Lực (M5 Setup)
* **Khung thời gian:** `InpSetupTF` (Mặc định: PERIOD_M5).
* **Quét Thanh Khoản (Liquidity Sweep):**
  * Tự động dò tìm các vùng đỉnh bằng nhau (Equal Highs - EQH) hoặc đáy bằng nhau (Equal Lows - EQL) nơi tập trung thanh khoản dừng lỗ của đám đông (Buy-side / Sell-side Liquidity Pools).
  * Nến Setup thò râu quét qua bể thanh khoản nhưng đóng cửa quay ngược trở lại bên trong vùng (Bẫy giá bẫy thanh khoản thành công).
* **Nến Bung Xung Lực (Displacement) & Phá Vỡ Cấu Trúc (True BOS):**
  * Nến bung xung lực phải có thân nến chiếm $\ge 65\%$ biên độ nến (`InpMinDisplacementBodyRatio`).
  * Biên độ nến phải gấp $\ge 1.25$ lần biên độ nến trung bình thuần OHLC (`GetAverageCandleRange`).
  * Nến phải đóng cửa vượt qua Swing Pivot đối diện (True Pivot High cho BUY hoặc True Pivot Low cho SELL) để xác nhận chuyển pha cấu trúc thị trường (BOS).
* **Tạo Vùng Mất Cân Bằng (Fair Value Gap - FVG):**
  * Xác định khoảng trống thanh khoản giữa giá High của nến 3 và giá Low của nến 1 (với nến tăng), hoặc giữa giá Low của nến 3 và giá High của nến 1 (với nến giảm).
  * Tính điểm chính giữa cân bằng năng lượng: $\text{Consequent Encroachment (CE)} = 50\% \text{ FVG}$.

### 4. Tầng 3: Nhịp Hồi và Xác Nhận Chuyển Pha Vi Mô (M1 Precision Entry)
* **Khung thời gian:** `InpEntryTF` (Mặc định: PERIOD_M1).
* **Kiểm tra Giới hạn Vùng (Strict Bounding):** Hủy bỏ setup ngay lập tức nếu giá xuyên thủng ranh giới đối diện của FVG (FVG bị triệt tiêu).
* **Ngưỡng Hồi Tối Thiểu (`InpFvgRetraceRatio`):** Giá phải hồi sâu tối thiểu chạm mốc 50% CE của FVG.
* **Xác Nhận Micro MSS (`InpEntryConfirmation = CONFIRM_MICRO_MSS`):**
  * Trong nhịp hồi vi mô vào FVG, hệ thống theo dõi và xác nhận một Micro Pivot Lower High (cho BUY) hoặc Micro Pivot Higher Low (cho SELL).
  * Chỉ khi xuất hiện nến M1 đóng cửa dứt khoát phá vỡ Micro Pivot đó (`Micro BOS`), lệnh mới được kích hoạt tại giá thị trường (Ask/Bid).

### 5. Tầng 4: Điểm Dừng Lỗ Vô Hiệu Hóa (Structural Invalidation SL)
* Điểm SL được đặt ngay ngoài điểm cực trị của râu nến quét thanh khoản (Sweep Extreme), cộng thêm khoảng đệm an toàn `InpInvalidationBufferPts` (mặc định 20 points).
* Nếu thị trường phá vỡ mức giá này, toàn bộ luận điểm cá mập săn thanh khoản bị bác bỏ hoàn toàn.

### 6. Tầng 5: Quản Trị Mục Tiêu 3 Pha (3-Phase Milestone Management)
* **TP1 - Bể Thanh Khoản Nội Bộ (True Internal Liquidity):**
  Hệ thống quét tìm đỉnh/đáy Swing Pivot nội bộ thực tế gần nhất nằm giữa Entry và External Target. Khi chạm TP1:
  * Đóng một phần khối lượng (Mặc định 25%).
  * Dời Stop Loss về bảo toàn vốn (BreakEven hoặc mức khóa dương tối thiểu).
* **TP2 - Bể Thanh Khoản Ngoại Vi (Major External Liquidity):**
  Chạm đỉnh/đáy lớn HTF (M15 High/Low):
  * Đóng tiếp 25% khối lượng lệnh.
  * Kích hoạt chế độ Trailing Cấu trúc cho phần khối lượng còn lại.
* **Runner Vô Hạn (50% Khối lượng):**
  Hoàn toàn không đặt mức Take Profit cứng (`TP3..TP6 = 0.0`), cho phép vị thế thả nổi để ăn trọn các con sóng mở rộng 150 - 400 pips.

### 7. Tầng 6: Dời Dừng Lỗ Sau Entry (Post-Entry Market Structure Trailing)
* **Quy tắc cốt lõi:** Hệ thống **tuyệt đối không dời SL** theo các Swing Pivot đã tồn tại từ trước thời điểm mở lệnh.
* Lệnh được bảo vệ không gian dao động tự nhiên bằng Invalidation SL ban đầu cho đến khi thị trường tạo thành một **Swing Pivot Higher Low (BUY) hoặc Lower High (SELL) mới hình thành SAU THỜI ĐIỂM VÀO LỆNH (`barTime > posOpenTime`)**.
* Khi có Pivot mới xác nhận: SL lập tức được kéo lên nấp sau đáy/đỉnh cấu trúc mới đó. Vị thế chỉ thoát khi thị trường thực sự bẻ gãy cấu trúc xu hướng.

### 8. Cơ Chế Bền Vững Dữ Liệu (State Persistence) & Giữ Lệnh Qua Đêm Thứ Sáu
* **Lưu trữ Global Variables:** Toàn bộ dữ liệu vùng FVG, giá mục tiêu, trạng thái hồi, cực trị vi mô và mã vé lệnh được đồng bộ hóa tức thì vào Global Variables của terminal MT5. Khi khởi động lại terminal, chuyển đổi khung thời gian hoặc khởi động lại VPS, EA khôi phục nguyên vẹn trạng thái mà không bị gián đoạn.
* **Bảo vệ Vị thế Thứ Sáu (`InpHoldFridayForInstitutional = true`):** Chế độ Scalp tuân thủ đóng lệnh trước giờ đóng cửa cuối tuần để tránh khoảng trống giá (Gap), trong khi các vị thế Institutional Hold có mức đệm lợi nhuận được phép giữ lệnh xuyên tuần để phục vụ mục tiêu ăn trọn sóng lớn.

---

## III. HỆ THỐNG 5 BỘ FILE THIẾT LẬP (PRESET .SET) VÀ BẢNG SO SÁNH TOÀN DIỆN

Thư mục `file set/` cung cấp đầy đủ 5 cấu hình tối ưu sẵn sàng sử dụng, được phân chia chính xác theo từng phong cách giao dịch và khẩu vị rủi ro:

### 1. File Set 1: `Swing_Institutional_Hold_XAUUSD.set` (CHUẨN ĐỊNH CHẾ CÂN BẰNG - KHUYÊN DÙNG SỐ 1)
* **Mục tiêu:** Tối ưu hóa tỷ lệ R:R ($\ge 1:2.5$), entry sâu, lọc nhiễu tối đa, giữ sóng 100 - 300 pips.
* **Khung chart gắn EA:** **XAUUSD M5**.
* **Đặc tính kỹ thuật:**
  * `InpStrategyEngine = 1` (Institutional Swing Hold thuần Price Action 100%).
  * `InpPurePriceActionOnly = true` (Loại bỏ hoàn toàn chỉ báo ATR).
  * `InpUseHTFBias = true`, `InpFilterAgainstHTFBias = true` (Chặn tuyệt đối lệnh ngược cấu trúc H1).
  * `InpMinRoomRewardRatio = 2.50` (Room đến External Target phải gấp $\ge 2.5$ lần SL).
  * `InpRequireFvgRetest = true`, `InpFvgRetraceRatio = 0.50` (Chờ hồi sâu $50\%$ CE của FVG).
  * `InpEntryConfirmation = 3` (`CONFIRM_MICRO_MSS`: Nến M1 đóng cửa phá vỡ Micro Pivot).
  * `InpUseStructureTrailing = true` (Chỉ dời SL theo Pivot HL/LH sinh ra SAU ENTRY).
  * `InpHoldFridayForInstitutional = true` (Giữ vị thế Hold xuyên tuần).
  * `InpRiskPercent = 1.0%` (Rủi ro chuẩn mực cho tài khoản thực tế).
* **Tần suất:** 1 - 3 lệnh / ngày.

---

### 2. File Set 2: `Swing_Institutional_Aggressive_HighReturn.set` (MẠO HIỂM LỢI NHUẬN CAO - HOLD SÓNG LỚN)
* **Mục tiêu:** Tăng trưởng tài khoản thần tốc ($+50\% \to +120\%$/tháng), bắt trọn chân sóng bùng nổ, chấp nhận mức sụt giảm lớn hơn.
* **Khung chart gắn EA:** **XAUUSD M5**.
* **Đặc tính kỹ thuật:**
  * `InpRiskPercent = 2.5%` (Mỗi lệnh thắng 1:3 $\to$ 1:5 mang về $+7.5\% \to +12.5\%$ số dư tài khoản).
  * `InpFilterAgainstHTFBias = false` (Tắt chặn H1 Bias để bắt cả sóng đảo chiều đỉnh/đáy và sóng hồi sâu).
  * `InpMinRoomRewardRatio = 1.80` (Nới lỏng điều kiện Room, tăng gấp đôi số lượng cơ hội vào lệnh).
  * `InpFvgRetraceRatio = 0.40` (Chỉ cần retrace 40% FVG là bắt đầu canh entry, tránh lỡ kèo nông).
  * `InpEntryConfirmation = 2` (`CONFIRM_MICRO_M1_CLOSE`: Nến M1 rút râu là khớp lệnh ngay ở chân sóng).
  * `InpTP1_ClosePercent = 20%`, `InpTP2_ClosePercent = 20%`, **thả nổi 60% Runner** tối đa hóa lợi nhuận.
* **Tần suất:** 2 - 5 lệnh / ngày.

---

### 3. File Set 3: `Swing_Institutional_PropFirm_Conservative.set` (BẢO TỒN VỐN NGHIÊM NGẶT / THI QUỸ)
* **Mục tiêu:** Bảo vệ tài khoản tuyệt đối, tuân thủ 100% quy định Drawdown ngày của các quỹ đầu tư (FTMO, MFF, The5ers).
* **Khung chart gắn EA:** **XAUUSD M5**.
* **Đặc tính kỹ thuật:**
  * `InpRiskPercent = 0.5%` (Rủi ro tối đa 0.5% số dư trên mỗi lượt giao dịch).
  * `InpMinRoomRewardRatio = 3.00` (Chỉ chấp nhận các setup có tiềm năng lợi nhuận gấp $\ge 3.0$ lần rủi ro).
  * `InpMaxRetestWaitBars = 6` (Nếu quá 6 nến M5 không retest thì tự hủy setup, bảo đảm tính tươi mới).
  * `InpMaxDailyLossPercent = 3.0%` (Chạm mức sụt giảm 3% trong ngày là tự động ngắt giao dịch).
  * `InpCooldownMinutesAfterLoss = 30` (Nghỉ 30 phút sau mỗi lệnh thua để tránh giao dịch trả thù).
* **Tần suất:** 0.5 - 2 lệnh / ngày.

---

### 4. File Set 4: `Scalp_Ultra_HighFrequency_Aggressive.set` (SCALPING SIÊU TỐC TẦN SUẤT CAO - BẮN LỆNH TICK)
* **Mục tiêu:** Lướt sóng chớp nhoáng theo xung lực từng Tick giá, quay vòng vốn liên tục 30 - 80 lệnh/ngày.
* **Khung chart gắn EA:** **XAUUSD M1**.
* **Đặc tính kỹ thuật:**
  * `InpStrategyEngine = 0` (Scalping M1+M5 EMA).
  * `InpExecutionTiming = 1` (`EXEC_INTRABAR_TICK`: Bắn lệnh siêu tốc từng Tick giá, không chờ đóng nến).
  * `InpSignalMode = 0` (Bắt cả Giao cắt EMA lẫn nhịp hồi Retest).
  * `InpRiskPercent = 2.0%` mỗi lệnh.
  * `InpTP1_R = 0.70R` (~7 - 12 pips Vàng) chốt ngay **70% khối lượng** giải phóng vị thế nhanh chóng.
  * `InpUseTimeStop = true`, `InpMaxHoldMinutes = 7` (Sau 7 phút không cắn TP thì tự đóng lệnh xoay vốn).
  * Chỉ giao dịch trong khung giờ biến động mạnh phiên Âu/Mỹ (13:00 - 22:00).
* **Tần suất:** 30 - 80 lệnh / ngày.

---

### 5. File Set 5: `Scalp_M5_M1_Confluence_AutoLot.set` (SCALPING M1+M5 ĐỒNG THUẬN KÉP CÂN BẰNG)
* **Mục tiêu:** Giao dịch lướt sóng hàng ngày cân bằng, Winrate 80%+, kiểm soát rủi ro chặt chẽ.
* **Khung chart gắn EA:** **XAUUSD M5** hoặc **M1**.
* **Đặc tính kỹ thuật:**
  * `InpStrategyEngine = 0` (Scalping).
  * `InpMtfMode = 1` (Bắt buộc đồng thuận xu hướng giữa M1 và M5).
  * `InpUseBaselineTrend = true` (Bộ lọc xu hướng nền tảng EMA 200).
  * `InpAllowReverseClose = true` (Đảo chiều tức thì khi cả M1 và M5 đồng loạt chuyển hướng ngược lại).
  * `InpUseMidWayBE = true` (Dời SL lên dương khi giá chạy được 50% quãng đường tới TP1).
  * `InpRiskPercent = 1.5% - 2.0%`.
* **Tần suất:** 6 - 15 lệnh / ngày.

---

### BẢNG SO SÁNH TỔNG HỢP 5 BỘ FILE PRESET (.SET)

| Tiêu chí kỹ thuật | Swing_Hold_XAUUSD (Chuẩn) | Swing_Aggressive (Mạo hiểm) | Swing_PropFirm (Bảo thủ) | Scalp_Ultra_Fast (Siêu tốc) | Scalp_M5_M1 (Cân bằng) |
| :--- | :---: | :---: | :---: | :---: | :---: |
| **Động cơ cốt lõi** | **Institutional Hold** | **Institutional Hold** | **Institutional Hold** | **Scalp EMA M1** | **Scalp EMA M1+M5** |
| **Cơ chế phân tích** | **100% Pure Price Action**| **100% Pure Price Action**| **100% Pure Price Action**| Dải động lượng EMA/Tick | Đồng thuận EMA + EMA 200 |
| **Chart gắn EA** | **M5** | **M5** | **M5** | **M1** | **M5 hoặc M1** |
| **Mức rủi ro / lệnh** | **1.0%** | **2.5%** | **0.5%** | **2.0%** | **1.5% - 2.0%** |
| **Tần suất lệnh / ngày** | **1 - 3 lệnh** | **2 - 5 lệnh** | **0.5 - 2 lệnh** | **30 - 80 lệnh** | **6 - 15 lệnh** |
| **Tỷ lệ R:R tối thiểu** | **1:2.5** | **1:1.8** | **1:3.0** | 1:0.7 (Scalp) | 1:1.2 (Scalp) |
| **Xác nhận điểm vào** | Micro MSS (M1 BOS) | Nến M1 rút râu (Vào sớm) | Micro MSS (M1 BOS) | Từng Tick giá (Intrabar) | Đóng nến Retest EMA |
| **Lọc cấu trúc H1** | Bắt buộc (Chặn ngược trend)| Tắt chặn (Bắt cả đảo chiều)| Bắt buộc (Siêu nghiêm ngặt)| Không áp dụng | Lọc EMA 200 |
| **Quản lý TP / Runner** | TP1(25%) - TP2(25%) - Run(50%)| TP1(20%) - TP2(20%) - Run(60%)| TP1(25%) - TP2(25%) - Run(50%)| TP1(70%) - TP2(20%) - Run(10%)| TP1(30%) - TP2(30%) - Run(40%)|
| **Cơ chế Trailing SL** | Post-Entry Structure Trailing| Post-Entry Structure Trailing| Post-Entry Structure Trailing| ATR Dynamic Trailing | ATR Dynamic Trailing |
| **Giữ lệnh qua đêm T6** | Cho phép | Cho phép | Cho phép | Tự động đóng (21:00) | Tự động đóng (21:00) |
| **Mức sụt giảm tối đa (DD)**| Thấp ($< 8\%$) | Trung bình ($12\% - 18\%$) | Cực thấp ($< 4\%$) | Biến thiên theo Spread | Thấp ($< 6\%$) |
| **Khẩu vị nhà đầu tư** | **Đầu tư chuẩn định chế** | **Tăng trưởng vốn thần tốc**| **Thi quỹ, vốn lớn** | **Giao dịch lướt sóng ngắn**| **Scalping hàng ngày** |

---

## IV. BẢNG PHÂN BỔ VỐN VÀ ĐỘ AN TOÀN TÀI KHOẢN (MONEY MANAGEMENT)

Trên sàn giao dịch tiêu chuẩn MT5 đối với XAUUSD:
* 1.00 lot tiêu chuẩn = 100 ounces vàng.
* 0.01 lot tối thiểu = 1 ounce vàng.
* Biến động $1.00 USD (10 pips = 100 points) trên 0.01 lot = $1.00 USD lãi/lỗ.

| Mức Vốn (USD) | Loại Tài Khoản | Khối Lượng Khuyên Dùng | Đánh Giá Mức Độ Chịu Đựng Sụt Giảm |
| :---: | :---: | :---: | :--- |
| **$20 - $50** | Cent (USC) | `0.01 - 0.02 lot Cent` | **Bất tử 100%:** 2,000 - 5,000 Cent. Lỗ 1 lệnh chỉ mất $0.03 - $0.05 USD. Chịu được chuỗi 100 lệnh thua liên tiếp. |
| **$200 - $300** | Chuẩn (Standard/ECN) | `0.01 lot cố định` | **Rất an toàn:** Mỗi lệnh thua trung bình $2.50 - $4.00 USD (chỉ chiếm 1.0% - 1.5% tài khoản). |
| **$500 - $1,000** | Chuẩn (Pro/Zero Spread)| `InpRiskPercent = 1.0%` | **Tối ưu hóa lợi nhuận:** Lot tự động tính theo khoảng cách SL thực tế để duy trì đúng 1.0% rủi ro. |
| **$\ge $10,000** | Tài khoản Quỹ / VIP | `InpRiskPercent = 0.5%` | **Bảo tồn vốn tổ chức:** Tuân thủ chuẩn mực Drawdown của quỹ đầu tư chuyên nghiệp. |

---

## V. QUY TRÌNH TRIỂN KHAI TRÊN METATRADER 5

1. Mở phần mềm **MetaTrader 5**.
2. Chọn menu **Tệp (File)** $\to$ chọn **Mở Thư mục Dữ liệu (Open Data Folder)**.
3. Sao chép toàn bộ thư mục EA vào đường dẫn `MQL5\Experts\`.
4. Mở biểu đồ **XAUUSD** ở khung thời gian **M5** (đối với Swing Hold) hoặc **M1** (đối với Scalp).
5. Kéo thả file `Sniper_EA_V4_Production.ex5` vào biểu đồ.
6. Tại thẻ **Inputs**, chọn **Load (Nạp)** và duyệt tới 1 trong 5 file cấu hình tương ứng trong thư mục `file set/`:
   * `Swing_Institutional_Hold_XAUUSD.set` (Chuẩn Định Chế cân bằng - Khuyên dùng số 1)
   * `Swing_Institutional_Aggressive_HighReturn.set` (Mạo hiểm lợi nhuận cao - Hold sóng lớn)
   * `Swing_Institutional_PropFirm_Conservative.set` (Bảo tồn vốn nghiêm ngặt / Thi quỹ)
   * `Scalp_Ultra_HighFrequency_Aggressive.set` (Scalping siêu tốc từng Tick giá)
   * `Scalp_M5_M1_Confluence_AutoLot.set` (Scalping M1+M5 đồng thuận kép cân bằng)
7. Đảm bảo nút **Algo Trading** trên thanh công cụ của MT5 đã được bật (hiển thị màu xanh lá cây).
8. Quan sát Dashboard góc trên bên phải màn hình: Xác nhận dòng `H1 Structure` và trạng thái `Hold Engine: SCANNING`.

---

## VI. BẢN QUYỀN VÀ KHUYẾN CÁO MIỄN TRỪ TRÁCH NHIỆM

Hệ thống được phát triển bởi **Nguyễn Quang Tú (qtusdev)**.

Thị trường tài chính phái sinh, đặc biệt là giao dịch Vàng (XAUUSD) có sử dụng đòn bẩy, luôn tiềm ẩn rủi ro biến động giá rất cao. Người dùng cần hiểu rõ rằng hiệu suất trong quá khứ thông qua kiểm thử dữ liệu lịch sử (Backtest) không đảm bảo kết quả tương lai. Bắt buộc phải thực hiện kiểm thử trên tài khoản Demo hoặc tài khoản Cent tối thiểu 2 - 4 tuần trước khi vận hành trên tài khoản vốn thực tế. Tác giả không chịu trách nhiệm cho bất kỳ tổn thất tài chính nào phát sinh từ quyết định giao dịch của người dùng.
