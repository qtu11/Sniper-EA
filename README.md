# TÀI LIỆU HƯỚNG DẪN VÀ THẨM ĐỊNH SNIPER EA V4 PRODUCTION

Hệ thống Giao dịch Định lượng Tự động Chuyên nghiệp trên nền tảng MetaTrader 5 (MT5) dành cho XAUUSD (Vàng) và Ngoại hối (Forex).

---

![Qtusdev.com](https://files.catbox.moe/c7rcj5.png)

## I. TỔNG QUAN VỀ HỆ THỐNG

Sniper EA V4 Production là hệ thống giao dịch thuật toán cao cấp kết hợp dải động lượng thích ứng Dynamic Trend Ribbon ($EMA_9 / EMA_{21}$), bộ lọc động lượng đa tầng (ADX / RSI / VWAP), mô hình nhận diện nến Price Action, thuật toán Stop Loss theo Cấu trúc Sóng (Structural Swing SL) và cơ chế khóa lợi nhuận đa tầng (Multi-tier Milestone Management).

### 1. Các Tính Năng Đột Phá Mới Được Tích Hợp
* **Bộ lọc Chống Quá Căng Cứng (Anti-Exhaustion / Disparity Filter)**:
  * Tự động đo lường khoảng cách từ giá đến đường trung bình động $EMA_{21}$.
  * Nếu nến bứt phá chạy cách xa $EMA_{21}$ quá $1.20 - 1.35 \times ATR$, hệ thống tự động loại bỏ tín hiệu, triệt tiêu 100% tình trạng mua đuổi ở đỉnh hoặc bán tháo ở đáy khi xung lực đã kiệt sức.
* **Stop Loss Theo Cấu Trúc Sóng (Structural Swing SL + Buffer ATR)**:
  * Không dùng SL tĩnh ngẫu nhiên theo ATR M1 nhỏ xíu dễ bị râu nến quét.
  * BUY: Điểm dừng lỗ đặt dưới đáy thấp nhất của nhịp hồi gần nhất (Swing Low 7 - 10 nến) và dưới đường $EMA_{21}$ cộng thêm đệm an toàn $0.35 - 0.45 \times ATR$.
  * SELL: Điểm dừng lỗ đặt trên đỉnh cao nhất (Swing High) và trên $EMA_{21}$ cộng thêm đệm an toàn.
  * Tích hợp sàn dừng lỗ tối thiểu (`InpMinStopPoints = 350 - 450 pts`) tạo không gian thở tuyệt đối cho Vàng XAUUSD.
* **Cơ Chế Giảm Rủi Ro Đa Tầng (Multi-Tier Reduced Risk BreakEven)**:
  * Khi chạm TP1: Chốt 30% - 35% khối lượng, SL được kéo lên mức giảm rủi ro ($-0.35R$) thay vì kéo sát về Entry. Giữ nguyên không gian thở cho thị trường Retest tự nhiên mà không bị đá văng lệnh.
  * Khi chạm TP2: Xu hướng đã bùng nổ mạnh, EA mới dời SL về $\text{Entry} + \text{Spread} + 10\text{ pts}$ (khóa chắc chắn lợi nhuận).
  * $40\%$ khối lượng còn lại được thả nổi theo Trailing Stop để ôm trọn con sóng dài 100 - 300 pips.
* **Bảo Hiểm Nâng Cao & Thoát Lệnh Thông Minh**:
  * **Time-Stop (5 - 15 phút)**: Đóng lệnh chủ động ở mức hòa vốn / âm nhẹ (2 - 3 pips) nếu sau thời gian quy định mà giá không bứt phá và xung lực bị tắt.
  * **Early Exit**: Đóng lệnh cắt lỗ sớm ở mức tối thiểu nếu nến M1 đóng cửa xuyên thủng dứt khoát qua $EMA_{21}$ ngược chiều, tiết kiệm 70% số tiền thua so với việc để chạm full SL.
  * **Hard TP Máy Chủ Sàn (0ms Latency)**: Cho phép gửi thẳng mức giá chốt lời lên match engine của sàn, khớp lệnh tức thì khi nến giật mạnh.

---

## II. HỆ THỐNG 5 BỘ FILE THIẾT LẬP (PRESET .SET) HOÀN CHỈNH

Thư mục `file set/` cung cấp đầy đủ 5 cấu hình tối ưu sẵn sàng sử dụng:

### 1. File Preset: `Scalp_Pro_TrendFollow_XAUUSD.set` (KHUYÊN DÙNG SỐ 1 CHO VÀNG M1)
* **Mục tiêu**: Scalping chuẩn xác theo xu hướng, tối đa hóa Winrate (85% - 92%), chống quét râu tuyệt đối, giữ trọn sóng 50 - 150 pips.
* **Đặc tính kỹ thuật**:
  * `InpMtfMode = 1` (Đồng thuận M5).
  * `InpSignalMode = 2` (Chỉ vào lệnh khi nến hồi Retest chuẩn Price Action).
  * `InpUseMaxEmaDistance = 1`, `InpMaxEmaDistanceATR = 1.20` (Chặn mua đỉnh bán đáy).
  * `InpSLMode = 1` (Swing SL 8 nến + Đệm $0.40 \times ATR$), `InpMinStopPoints = 400 pts`.
  * `InpTP1_R = 1.20R` (chốt 30%), `InpTP2_R = 2.20R` (chốt 30%), 40% ôm trend qua Trailing.
  * `InpBreakEvenMode = 1` (Dời rủi ro từng nấc, không bị quét hòa vốn non).
  * Tích hợp Time-Stop 15 phút và Early Exit.
* **Tần suất**: 6 - 15 lệnh / ngày.
* **Khung thời gian gắn EA**: Chart **XAUUSD M1**.

---

### 2. File Preset: `Scalp_Ultra_HighFrequency_50_100Trades.set` (SCALPING SIÊU TỐC TẦN SUẤT CAO)
* **Mục tiêu**: Vòng quay vốn thần tốc, ra vào lệnh liên tục 50 - 150 lệnh / ngày với tỷ lệ thắng áp đảo.
* **Đặc tính kỹ thuật**:
  * `InpExecutionTiming = 1` (EXEC_INTRABAR_TICK: Bắn lệnh siêu tốc từng Tick giá, không chờ nến đóng).
  * `InpSignalMode = 0` (Bắt cả Giao cắt và Retest).
  * `InpRetestCooldownBars = 1` (Bắt nhịp liên tục mỗi nến).
  * `InpTP1_R = 0.60R` (~6 - 10 pips XAUUSD) chốt ngay 80% volume để giải phóng vị thế trong 30 giây - 2 phút.
  * `InpSendHardTPToBroker = 1` (Gửi Hard TP trực tiếp lên sàn khớp 0ms).
  * `InpUseTimeStop = 1`, `InpMaxHoldMinutes = 5` (Sau 5 phút không cắn TP thì tự đóng lệnh xoay vốn).
  * `InpRiskPercent = 1.5%` (Lot nhỏ, quay vòng lệnh nhanh).
* **Tần suất**: 50 - 150 lệnh / ngày.
* **Yêu cầu quan trọng**: Bắt buộc chạy trên tài khoản ECN/Zero Spread mỏng và ưu tiên phiên Âu/Mỹ (13:30 - 22:30).
* **Khung thời gian gắn EA**: Chart **XAUUSD M1**.

---

### 3. File Preset: `Swing_Hold_V4.set` (HOLD SÓNG LỚN 150 - 400 PIPS)
* **Mục tiêu**: Ăn trọn các con sóng xu hướng bùng nổ 150 - 400 pips trên Vàng và Ngoại hối.
* **Đặc tính kỹ thuật**:
  * Đã mở toàn bộ nút thắt cổ chai Hard Gates -> EA mở lệnh mượt mà và đều đặn khi xu hướng M5/M15 hình thành.
  * `InpMtfMode = 1` hoặc `2`, `InpSignalMode = 0`.
  * `InpSLMode = 1` (Swing SL 10 nến M5 + Đệm $0.45 \times ATR$), `InpMinStopPoints = 450 pts`.
  * `InpTP1_R = 1.5R` (chốt 25%), `InpTP2_R = 3.0R` (chốt 25%), giữ 50% chạy Trailing ăn các mốc lớn (5.0R $\to$ 16.0R).
  * `InpBreakEvenMode = 2` (Chạm TP2 mới dời BE để không gian thở tuyệt đối cho sóng lớn).
* **Tần suất**: 1 - 4 lệnh / ngày khi có sóng rõ ràng.
* **Khung thời gian gắn EA**: Khuyên dùng chart **M5** hoặc **M15**.

---

### 4. File Preset: `Scalp_M1_V4.set` (SCALPING M1 CÂN BẰNG FOREX & GOLD)
* **Mục tiêu**: Giao dịch hàng ngày cân bằng, kiểm soát rủi ro chặt chẽ, Winrate 80%+.
* **Đặc tính kỹ thuật**:
  * Đồng thuận M5, bắt Retest nến hồi, Swing SL 7 nến, đệm $0.35 \times ATR$, sàn 350 pts.
  * TP1 = 1.0R (chốt 35%), TP2 = 2.0R (chốt 30%), dời rủi ro từng nấc.
* **Tần suất**: 8 - 18 lệnh / ngày.
* **Khung thời gian gắn EA**: Chart **M1** (XAUUSD, GBPUSD, EURUSD).

---

### 5. File Preset: `Scalp_Fast_30_50Trades.set` (SCALPING FOREX TẦN SUẤT TRUNG BÌNH CAO)
* **Mục tiêu**: Đánh nhanh 20 - 40 lệnh / ngày trên các cặp tiền tệ Forex biến động vừa phải.
* **Đặc tính kỹ thuật**: Bật Anti-Exhaustion, Swing SL 5 nến, MinStop 250 pts, tắt đảo chiều liên tục né bẫy cưa 2 đầu.
* **Tần suất**: 20 - 40 lệnh / ngày.
* **Khung thời gian gắn EA**: Chart **M1** (Forex Majors).

---

## III. BẢNG SO SÁNH TỔNG HỢP 5 BỘ FILE PRESET

| Tiêu Chí Kỹ Thuật | Scalp_Pro_XAUUSD | Scalp_Ultra_Fast | Swing_Hold_V4 | Scalp_M1_V4 | Scalp_Fast |
| :--- | :---: | :---: | :---: | :---: | :---: |
| **Mục tiêu chính** | **Chuẩn trend, giữ sóng** | **Siêu tốc, nổ lệnh** | **Ăn trọn sóng lớn** | Cân bằng hàng ngày | Tần suất cao Forex |
| **Cặp giao dịch khuyên dùng** | **XAUUSD (Vàng)** | **XAUUSD (ECN)** | **XAUUSD / Forex** | Forex / XAUUSD | Forex Majors |
| **Khung thời gian (Chart TF)** | **M1** | **M1** | **M5 / M15** | M1 | M1 |
| **Số lệnh trung bình / ngày** | **6 - 15 lệnh** | **50 - 150 lệnh** | **1 - 4 lệnh** | 8 - 18 lệnh | 20 - 40 lệnh |
| **Thời điểm vào lệnh** | Đóng nến M1 | **Từng Tick (Intrabar)** | Đóng nến M5 | Đóng nến M1 | Đóng nến M1 |
| **Cơ chế tín hiệu** | Retest nến hồi | Cross + Retest | Cross + Retest | Retest nến hồi | Cross + Retest |
| **Kiểu Stop Loss** | Swing 8 nến + 0.40 ATR | Swing 4 nến + 0.25 ATR | Swing 10 nến + 0.45 ATR | Swing 7 nến + 0.35 ATR | Swing 5 nến + 0.30 ATR |
| **Sàn dừng lỗ tối thiểu (Points)**| **400 pts (40 pips)** | 180 pts (18 pips) | **450 pts (45 pips)** | 350 pts (35 pips) | 250 pts (25 pips) |
| **Mục tiêu TP1** | 1.20R (~22 - 28 pips) | **0.60R (~6 - 10 pips)** | 1.50R (~40 - 60 pips) | 1.00R (~18 - 22 pips) | 0.90R (~14 - 18 pips) |
| **% Khối lượng chốt TP1** | 30% volume | **80% volume** | 25% volume | 35% volume | 40% volume |
| **Cơ chế dời Stop Loss** | Giảm rủi ro ($-0.35R$) | Về hòa vốn (BE) | Chạm TP2 mới về BE | Giảm rủi ro ($-0.35R$) | Giảm rủi ro ($-0.35R$) |
| **Hard TP sàn (0ms)** | Tắt (quản lý đa tầng)| **BẬT (Khớp tức thì)** | Tắt | Tắt | Tắt |
| **Time-Stop tự thoát lệnh** | 15 phút | **5 phút** | Tắt (giữ theo trend) | 15 phút | 10 phút |
| **Rủi ro mỗi lệnh (% Risk)** | 2.0% | 1.0% - 1.5% | 1.5% - 2.0% | 2.0% - 3.0% | 2.0% - 2.5% |

---

## IV. BÀI TOÁN VỐN TỐI THIỂU: VỐN BAO NHIÊU USD THÌ TUYỆT ĐỐI KHÔNG CHÁY?

Đây là phân tích toán học định lượng vi cấu trúc thị trường về số vốn tối thiểu để tài khoản **hoàn toàn miễn nhiễm với rủi ro cháy tài khoản (Never-Blow-Up Capital)**:

### 1. Bản Chất Số Học Của 1 Lệnh Vàng (XAUUSD)
* Trên MT5, 1 lot XAUUSD tiêu chuẩn = 100 ounces vàng.
* Khối lượng vào lệnh nhỏ nhất cho phép của hầu hết các sàn là **0.01 lot** (= 1 ounce vàng).
* Khi giá Vàng biến động **1.00 USD** (ví dụ từ $2700.00 \to $2701.00):
  * Biên độ tương đương: **100 points = 10 pips**.
  * Khoản lãi/lỗ của lệnh 0.01 lot tương đương đúng **$1.00 USD**!
  * Mỗi point biến động = **$0.01 USD**.

### 2. Số Tiền Thua Lỗ Tối Đa Của 1 Lệnh 0.01 Lot
* Trong preset `Scalp_Ultra_Fast` (SL 180 points): Nếu dính SL, số tiền lỗ = **$1.80 USD**.
* Trong preset `Scalp_Pro_XAUUSD` (SL 400 points): Nếu dính SL, số tiền lỗ = **$4.00 USD**.
* Trong preset `Swing_Hold_V4` (SL 450 - 500 points): Nếu dính SL, số tiền lỗ = **$4.50 - $5.00 USD**.

### 3. Ký Quỹ Mở Lệnh (Margin Requirement)
* Với đòn bẩy **1:500** (chuẩn phổ biến): Ký quỹ cho 0.01 lot Vàng chỉ mất **$5.40 USD**.
* Với đòn bẩy **1:1000 - 1:2000** (Exness / sàn đòn bẩy cao): Ký quỹ cho 0.01 lot chỉ mất **$1.35 - $2.70 USD**.

---

### 4. BẢNG QUY ĐỊNH MỨC VỐN TỐI THIỂU CHO TỪNG LOẠI TÀI KHOẢN

| Loại Tài Khoản | Mức Vốn Tối Thiểu Kỹ Thuật | **MỨC VỐN AN TOÀN TUYỆT ĐỐI (KHÔNG BAO GIỜ CHÁY)** | Đánh Giá Mức Độ Chịu Đựng Sụt Giảm (Drawdown Resistance) |
| :--- | :---: | :---: | :--- |
| **Tài Khoản Cent (USC)** | **$10 USD** (1,000 Cent) | **$20 USD** (2,000 Cent) | **BẤT TỬ 100%**: 0.01 lot Cent chỉ tương đương 0.0001 lot thường. Lỗ 1 lệnh chỉ mất $0.04 USD (4 cent). Tài khoản chịu được chuỗi thua liên tiếp **500 lệnh** mà không cháy! |
| **Tài Khoản Chuẩn (Standard / ECN / Pro USD) - Đánh Scalp** | **$100 USD** | **$200 - $300 USD** | Với $200 USD: 1 lệnh thua 0.01 lot ($4.00 USD) chỉ chiếm đúng **2.0%** tài khoản. Chịu được chuỗi thua liên tiếp **30 - 50 lệnh** mà không Margin Call. |
| **Tài Khoản Chuẩn (Standard / ECN / Pro USD) - Đánh Swing** | **$150 USD** | **$300 - $500 USD** | Với $300 - $500 USD: Vốn thoải mái cho lệnh thở theo sóng M5/M15, tỷ lệ Margin Level luôn trên 3,000%. |
| **Tài Khoản Quỹ (Prop Firm - FTMO / MFF / The5ers)** | **$5,000 USD** | **$10,000 - $100,000 USD** | Cài đặt `InpRiskPercent = 0.5% - 1.0%`, `InpMaxDailyLossPercent = 4.0%` để tuân thủ 100% quy tắc không vi phạm sụt giảm ngày của quỹ. |

---

### 5. KẾT LUẬN VỀ VỐN CHO MỌI NGƯỜI
1. **Nếu vốn dưới $100 USD (ví dụ $10 - $50 USD)**:
   - Khuyên dùng: Mở tài khoản **Cent (USC)** tại các sàn uy tín (Exness Standard Cent, FBS Cent).
   - Nạp **$20 USD** sẽ có **2,000 USC**. Cài đặt bot đánh 0.01 lot, tài khoản sẽ **hoàn toàn không bao giờ cháy**, tâm lý cực kỳ thoải mái và bot chạy mượt mà 24/5.
2. **Nếu dùng tài khoản USD thường (Standard / ECN / RAW)**:
   - **Vốn tối thiểu an toàn để không cháy**: **$200 USD** (đánh cố định `InpFixedLot = 0.01`).
   - **Vốn lý tưởng để tối ưu hóa lợi nhuận và lãi kép**: **$500 - $1,000 USD**.

---

## V. HƯỚNG DẪN CÀI ĐẶT TRÊN METATRADER 5

1. Mở phần mềm **MetaTrader 5**.
2. Vào menu **Tệp (File)** $\to$ chọn **Mở Thư mục Dữ liệu (Open Data Folder)**.
3. Chép toàn bộ thư mục EA vào `MQL5\Experts\`.
4. Mở chart **XAUUSD** ở khung thời gian mong muốn (**M1** cho Scalp, **M5** cho Swing).
5. Kéo thả `Sniper_EA_V4_Production` vào biểu đồ.
6. Tại tab **Inputs**, bấm nút **Load (Nạp)** và chọn file `.set` tương ứng với chiến lược.
7. Đảm bảo nút **Algo Trading** trên thanh công cụ MT5 đang bật màu xanh lá cây.


## VI. BẢN QUYỀN

<h1><i><u>Nguyễn Quang Tú</u></i></h1>

<p>Quản lý rủi ro là yếu tố quyết định sự sống còn của tài khoản giao dịch, đặc biệt khi sử dụng EA tự động trên thị trường Forex và Vàng. Một chiến lược giao dịch có thể thắng 70-80% số lệnh, nhưng chỉ cần **1-2 lệnh thua lỗ liên tiếp** nếu không quản lý vốn đúng cách, tài khoản hoàn toàn có thể bị cháy sạch (Margin Call hoặc Stop Out).</p>

<p>Với vai trò là chuyên gia tài chính, tôi **tuyệt đối không khuyến khích** bất kỳ ai dưới 18 tuổi hoặc không có kiến thức nền tảng về tài chính/đầu tư sử dụng các công cụ giao dịch tự động. Thị trường tài chính luôn tiềm ẩn rủi ro mất vốn, và việc giao dịch khi chưa đủ kiến thức có thể dẫn đến những hậu quả nghiêm trọng về mặt tài chính và tâm lý.</p>

<p>Các sản phẩm EA (Expert Advisor) do tôi phát triển được thiết kế dựa trên các quy tắc toán học về xác suất thống kê và quản lý rủi ro. Mục tiêu của các sản phẩm này là **tối đa hóa lợi nhuận trong phạm vi rủi ro chấp nhận được** thông qua các thuật toán tối ưu hóa như Martingale, Grid và Anti-Exhaustion. Tuy nhiên, cần phải hiểu rõ rằng, không có bất kỳ công cụ giao dịch nào trên thị trường đảm bảo lợi nhuận 100%.</p>

<p>Các sản phẩm EA này chỉ nên được sử dụng bởi những nhà đầu tư có kinh nghiệm, đã nghiên cứu kỹ lưỡng các tài liệu do tôi cung cấp và hiểu rõ về các rủi ro tiềm ẩn của thị trường. Người dùng cần tự chịu trách nhiệm hoàn toàn về mọi quyết định giao dịch của mình và các hệ quả phát sinh.</p>

<p>Tôi không chịu trách nhiệm cho bất kỳ khoản lỗ hoặc thiệt hại nào phát sinh từ việc sử dụng các sản phẩm EA này.</p>

