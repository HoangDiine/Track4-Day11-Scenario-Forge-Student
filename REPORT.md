MSSV: 202602751

# Lab 11 · Scenario Forge — báo cáo

Mỗi câu: trả lời ngắn, **trích số từ `evidence/`** (ghi tên file và trường, ví dụ `mc.json: crashes = 4`). Grader chạy lại với MSSV của bạn; số không khớp `evidence/` thì mất điểm phần D (RUBRIC), và câu dựa trên số sai khó được điểm đầy đủ. Không cần văn hay: một bảng nhỏ và hai, ba câu giải thích là đủ.

Thị trấn của bạn (`meta.json: _run.town`): **B**

## 1. Phát lại log có trung thực không? (bước A)

- **Số liệu từ `evidence/replay.json`**:
  - Số kịch bản bị loại do chính người lái gây va chạm: `dropped_human_crash_logs = 1` (trên tổng $n = 400$, số kịch bản hợp lệ sử dụng là `n_used = 399`).
  - Va chạm khi planner chạy trên log phát lại: `replay.crashes = 111` (trong đó `rear = 100`, `side = 11`, `front = 0`).
  - Va chạm trong thế giới phản ứng: `reactive.crashes = 1` (trong đó `side = 1`, `rear = 0`, `front = 0`).
  - Số kịch bản va chạm ở cả hai môi trường: `crash_in_both = 0`.

- **Giải thích cơ chế va chạm "giả"**:
  Phần lớn va chạm phát lại ($100/111$ vụ, chiếm hơn $90\%$) là va chạm từ phía sau (`rear`). Đây là các va chạm **hoàn toàn giả mạo (sim artifact)**: các phương tiện đi sau (như `TG`) trong log phát lại chỉ tua lại thụ động quỹ đạo ghi nhận sẵn của người lái. Khi planner mới phanh sớm hơn hoặc thận trọng hơn người lái (như giảm tốc trước góc khuất xe buýt), xe phía sau trong log không phản ứng nên đâm vào đuôi xe ta. Trong thế giới phản ứng (`reactive`), các xe sau lái theo IDM phản xạ với gia tốc phanh của xe ta và dừng an toàn, khiến số va chạm `rear` giảm về $0$.

- **Phân tích một kịch bản trong `replay_only`**:
  Tại kịch bản `scenario 0` (`replay.json: replay_only[0]`): trong log phát lại xảy ra va chạm `rear` do xe bám đuôi `TG` đâm đuôi xe ta; tuy nhiên trong thế giới phản ứng, min TTC giữa xe ta và xe sau là $7.46\text{ s}$ (`min_ttc_reactive = 7.46`), tình huống hoàn toàn an toàn do xe sau chủ động nhường khoảng cách.

---

## 2. Kịch bản bất khả thi (bước B)

- **Các quy tắc trong `is_feasible` và căn cứ vật lý**:
  1. *Gia tốc ngang khi chuyển làn*: $a_{\text{lat}} = (\pi / \text{cutDur})^2 \times 3.5 / 2 \le \mu g$. Quỹ đạo chuyển làn $3.5\text{ m}$ theo hàm cosine đạt đỉnh gia tốc ngang tại giữa quá trình; nếu vượt quá $\mu g$, lốp xe mất độ bám ngang và xe sẽ trượt văng khỏi mặt đường.
  2. *Gia tốc phanh sau nhập làn*: $\text{cutBrake} \le \mu g$. Lực phanh tối đa bị chặn bởi giới hạn ma sát trượt của mặt đường ($F = m a \le \mu m g$).
  3. *Tốc độ người đi bộ*: $\text{pedV} \le 2.5\text{ m/s}$ ($9\text{ km/h}$). Tốc độ đi bộ tự nhiên là $1.2 - 1.8\text{ m/s}$; trên $2.5\text{ m/s}$ là người chạy nước rút, không phải người đi bộ thông thường.
  4. *Tốc độ xe chuyển làn*: $0 < \text{cutV} \le 22.0\text{ m/s}$ ($79.2\text{ km/h}$). Vận tốc trong phố không thể âm hoặc vượt quá $22\text{ m/s}$.

- **Cuộc săn sinh nhiều kịch bản bất khả thi nhất (`evidence/hunt.json`)**:
  Cuộc săn `crosswalk_block` sinh nhiều kịch bản bất khả thi nhất với `infeasible = 12` (theo sau là `rear_ended` với `11` kịch bản, trong khi `bus_bypass` và `cut_in` chỉ có `2`).
  - *Nguyên nhân CEM bị kéo về phía bất khả thi*: Các tham số cực đoan phi vật lý (như người đi bộ lao qua đường cực nhanh hoặc xe phanh cháy lốp) tạo ra điểm nguy hiểm (`score`) cao nhất. Do thuật toán CEM tối ưu hóa theo score, nó tự nhiên bị hút về các vùng biên vi phạm vật lý này.
  - *Hậu quả nếu đếm kịch bản bất khả thi*: Xác suất rủi ro của planner sẽ bị thổi phồng nghiêm trọng (overestimation), báo cáo sai lệch những nguy cơ không thể xảy ra ngoài đời thực.

---

## 3. Monte Carlo cho bạn biết gì (bước C1)

- **Số liệu từ `evidence/mc.json`**:
  - Cỡ mẫu: $n = 3000$.
  - Số vụ va chạm: $k = 4$.
  - Ước lượng điểm: $\hat{p} = 0.001333$ ($1.33 \times 10^{-3}$).
  - Khoảng tin cậy Wilson 95%: $[0.000519, 0.003424]$ ($[5.19 \times 10^{-4}, 3.42 \times 10^{-3}]$).

- **Số lần chạy để khoảng tin cậy hẹp còn $\pm 20\%$ của $p$**:
  Với độ chính xác tương đối $\delta = 0.20$, cỡ mẫu lý thuyết cần thiết là $n \approx \frac{1.96^2}{\delta^2} \frac{1-p}{p} \approx \frac{96}{p}$.
  Với $\hat{p} = 1.333 \times 10^{-3}$, ta cần:
  $$n \approx \frac{96}{0.001333} \approx 72,000\text{ lần chạy}$$

- **Kiểu thất bại chưa xuất hiện trong 3 000 lần chạy**:
  Trong `mc.json: per_mode`, cả 4 kiểu `cut_in`, `crosswalk_block`, `rear_ended` và `other` đều có $k = 0$ (chỉ có `bus_bypass` xuất hiện 4 lần).
  - *Con số đúng cần báo*: **Không bao giờ được báo xác suất bằng 0**. Với $k = 0$, cận trên khoảng tin cậy Wilson 95% là **$0.001279$ ($1.28 \times 10^{-3}$)** (hoặc quy tắc số ba $3/n = 3/3000 = 1.0 \times 10^{-3}$). Con số cần báo là khoảng $[0, 1.28 \times 10^{-3}]$.

---

## 4. Importance sampling, ESS và độ suy biến (bước C3)

- **Bảng tổng hợp từ `evidence/estimate.json`**:

| Kiểu thất bại | Hạt giống (Seed) | $p$ | hits | $ess$ | top_share |
|---|---|---|---|---|---|
| `bus_bypass` | seed 0 | $6.17 \times 10^{-3}$ | 71 | 121.5 | 0.852 |
| `bus_bypass` | seed 1 | $5.80 \times 10^{-4}$ | 60 | 123.0 | 0.221 |
| `bus_bypass` | seed 2 | $7.59 \times 10^{-4}$ | 67 | 119.5 | 0.134 |
| `cut_in` | seed 0, 1, 2 | 0.0 (upper: 0.025) | 0 | 119.5 - 133.3 | 0.0 |
| `crosswalk_block` | seed 0 | $5.80 \times 10^{-7}$ | 98 | 119.0 | 0.259 |
| `crosswalk_block` | seed 1 | $1.21 \times 10^{-6}$ | 100 | 116.0 | 0.677 |
| `crosswalk_block` | seed 2 | $1.24 \times 10^{-5}$ | 100 | 119.1 | 0.765 |
| `rear_ended` | seed 0, 1, 2 | 0.0 (upper: 0.025) | 0 | 111.0 - 127.0 | 0.0 |
| **Tổng (`total`)** | seed 0 | $6.17 \times 10^{-3}$ | - | - | CI: $[0.0, 1.78 \times 10^{-2}]$ |
| **Tổng (`total`)** | seed 1 | $5.81 \times 10^{-4}$ | - | - | CI: $[2.28 \times 10^{-4}, 2.21 \times 10^{-3}]$ |
| **Tổng (`total`)** | seed 2 | $7.71 \times 10^{-4}$ | - | - | CI: $[3.75 \times 10^{-4}, 2.45 \times 10^{-3}]$ |

- **Độ đồng thuận giữa các seed và với Monte Carlo**:
  - Seed 1 ($5.81 \times 10^{-4}$) và Seed 2 ($7.71 \times 10^{-4}$) rất đồng nhất và nằm trọn trong khoảng Wilson 95% của Monte Carlo ($[5.19 \times 10^{-4}, 3.42 \times 10^{-3}]$).
  - Seed 0 ước lượng cao hơn khoảng 8-10 lần ($6.17 \times 10^{-3}$) do hiện tượng suy biến trọng số mẫu.

- **Hiện tượng `top_share` lớn (> 0.5)**:
  - Xuất hiện tại Seed 0 (`bus_bypass`, top_share = 0.852), Seed 1 (`crosswalk_block`, top_share = 0.677), và Seed 2 (`crosswalk_block`, top_share = 0.765).
  - Khi `top_share > 0.5`, hơn một nửa giá trị kỳ vọng bị chi phối bởi duy nhất một mẫu có trọng số $w = p/q$ cực lớn rơi vào vùng đuôi của đề xuất. Lúc này con số báo cáo thực chất là trọng số của một cá thể riêng lẻ thay vì phản ánh tần suất trung bình của phân phối.

- **So sánh ngân sách (`budget`)**:
  Ngân sách IS là `16,800` lượt chạy (`estimate.json: budget`). Với ngân sách này, IS tìm được các biến cố hiếm cỡ $10^{-6}$ (`crosswalk_block`). Nếu dùng Monte Carlo thô để đạt cùng độ chính xác cho biến cố $10^{-6}$, cần khoảng $\approx 96 / 10^{-6} \approx 96,000,000$ lần chạy. IS giúp tiết kiệm hơn $5,000$ lần ngân sách.

- **Nguồn gốc phần `other` và lý do không săn**:
  Phần `other` lấy từ `mc.json` ($k = 0, n = 3000, p = 0.0$, cận trên $hi = 0.001279$). Không săn `other` vì đây là tập hợp các va chạm linh tinh không có mô hình hình học thống nhất; việc cố gắng tối ưu hóa một hàm score không đồng nhất sẽ làm thuật toán CEM bị phân mảnh.

---

## 5. Các kiểu thất bại (bước C2 + 3D)

- **Các kiểu tìm thấy trong `evidence/hunt.json`**:
  1. `bus_bypass` (tìm thấy 221 vụ):
     - *Chuỗi sự kiện*: Xe buýt dừng đỗ bên lề tại $x = 326\text{ m}$ tạo điểm mù che khuất tầm nhìn. Xe cắt làn `C2` lách qua xe buýt ở tốc độ thấp ($v < 6\text{ m/s}$) và nhập vào làn xe ta. Xe ta sau khi vượt qua nút giao đã tăng tốc hành trình nên phát hiện muộn và đâm vào sườn `C2`.
     - *Ảnh 3D*: `evidence/shots/ch9-lab-bus-bypass.png` (B-04 trong `evidence/viewer.json`; va chạm với xe cắt làn lúc 12,5 s).
  2. `crosswalk_block` (tìm thấy 58 vụ):
     - *Chuỗi sự kiện*: Người đi bộ `P1` bước ra từ sau xe buýt để qua đường tại vạch kẻ $x = 122\text{ m}$. Planner phát hiện muộn, phanh gấp khẩn cấp và dừng hẳn lại, nhưng phần đầu thân xe dừng lấn lên vạch kẻ đường chặn lối đi của `P1`, buộc người đi bộ phải dừng bước.
     - *Ảnh 3D*: `evidence/shots/ch9-lab-crosswalk-block.png` (C-02 trong `evidence/viewer.json`; xe chặn vạch qua đường lúc 20,6 s).

- **Kiểu cuộc săn không tìm thấy**:
  Cuộc săn `cut_in` và `rear_ended` không tìm thấy va chạm đúng kiểu trong các vòng elite (tìm thấy các biến thể thuộc `bus_bypass` hoặc `crosswalk_block`). Độ tin cậy ở mức vừa phải: trong thị trấn B, mô hình IDM của planner giữ khoảng cách đuôi đủ an toàn và xe cắt làn ở tốc độ cao hiếm khi ép va chạm thành công ngoài khu vực góc khuất xe buýt.

- **Hậu quả nếu chỉ chạy MỘT cuộc săn chung**:
  Thuật toán CEM sẽ bị co cụm (mode collapse) về duy nhất kiểu va chạm phổ biến và dễ tìm nhất là `bus_bypass` (chiếm ưu thế với 221 vụ). Các kiểu thất bại quan trọng khác nhưng ở vùng không gian hẹp hơn như `crosswalk_block` sẽ bị đào thải khỏi nhóm elite và hoàn toàn không bao giờ được phát hiện.

---

## 6. Sửa planner (bước E + 3D)

- **Số liệu từ `evidence/fix.json`**:
  - **Policy được chọn**: `{'cutinLat': 3.3}`
  - **Trên tập TRAIN**:
    - Bản sửa: `fail = 1.34e-07`, `fail_hi = 2.22e-07`, `progress = 214.8 m` (loại `unseen: ['bus_bypass']`).
    - Baseline (`fix.json: baseline_train`): `fail = 1.71e-03`, `fail_hi = 3.33e-03`, `progress = 214.8 m`.
  - **Trên tập TEST độc lập**:
    - Bản sửa: `TEST fail = 1.55e-06`, `fail_hi_seen = 3.59e-06`, `fail_hi (valid) = 3.88e-02`, `unseen = ['bus_bypass']`, `progress = 214.0 m`, `harsh = 0.05`.
    - Baseline: `TEST fail = 1.18e-03`, `fail_hi_seen = 2.56e-03`, `fail_hi (valid) = 3.84e-03`, `unseen = []`, `progress = 214.3 m`, `harsh = 0.05`.

- **Vì sao `fail_hi` của bản sửa có thể lớn hơn `fail` của policy gốc?**:
  Bản sửa `{'cutinLat': 3.3}` triệt tiêu hoàn toàn va chạm `bus_bypass` trên tập TEST ($0$ va chạm sau 8 vòng re-hunt). Do không thấy va chạm nào, luật kiểm định thống kê trung thực bắt buộc phải cộng cận trên không quan sát $3.0 / (\beta n) = 3.0 / (0.2 \times 400) = 0.0375$ vào `fail_hi`. Do đó, `fail_hi` hợp lệ là $0.0388$, lớn hơn `fail` của baseline ($1.18 \times 10^{-3}$), mặc dù trên thực tế dữ liệu quan sát được (`fail_hi_seen = 3.59e-06`) an toàn hơn baseline gần 1.000 lần.

- **Núm chỉnh "nghe an toàn hơn" nhưng làm tăng thất bại**:
  Trong bảng TRAIN, ứng viên giảm vận tốc thận trọng `{'cautionV': 6.0}` có `fail = 8.54e-03` (tăng gấp 10 lần so với baseline $7.97 \times 10^{-4}$); ứng viên tăng khoảng cách an toàn `{'T': 2.0}` có `fail = 6.81e-03`.
  - *Cơ chế phản ứng của thế giới*: Khi xe ta giảm tốc độ quá sớm hoặc chừa khoảng cách đầu xe quá lớn, các phương tiện khác ở làn bên (`C2`) coi đó là khoảng trống an toàn và lập tức chuyển làn tạt đầu cắt mặt, tạo ra tình huống xung đột nguy hiểm hơn nhiều.

- **Hình ảnh 3D có bóng mờ**:
  `evidence/shots/ch9-lab-bus-bypass.png` cho thấy điểm B-04 trong `evidence/viewer.json`: policy đang xem va chạm lúc 12,5 s, còn policy gốc (bóng mờ) không va chạm. Đây là minh họa một ca cụ thể; các con số TRAIN/TEST ở trên vẫn lấy từ `evidence/fix.json`.

---

## 7. Từ mô phỏng ra đường thật

Ba giả định then chốt trong `avsim/space.py`:

1. **Phân phối khe hở chấp nhận nhập làn (`cut_gap_median = 12.0 m`)**:
   - *Dữ liệu thay thế*: Thu thập từ quỹ đạo thực tế qua camera/LiDAR roadside tại các điểm nghẽn đô thị (như bộ dữ liệu HighD, Argoverse).
   - *Hướng sai lệch*: Người lái xe thực tế tại các đô thị đông đúc thường chấp nhận các khe hở hẹp hơn ($4 - 8\text{ m}$). Giả định $12\text{ m}$ làm mô phỏng đánh giá thấp tần suất các vụ tạt đầu nguy hiểm.

2. **Xác suất người đi bộ quan sát xe qua xe buýt (`pedLook = 0.85`)**:
   - *Dữ liệu thay thế*: Khảo sát thị giác máy tính về hành vi người đi bộ tại các trạm dừng xe buýt thực tế.
   - *Hướng sai lệch*: Tỷ lệ người đi bộ mất tập trung (nhìn điện thoại) hoặc trẻ em không chú ý quan sát có thể lên tới $40-50\%$. Giả định $85\%$ người đi bộ đều quan sát làm mô phỏng đánh giá quá lạc quan độ an toàn trước vạch qua đường.

3. **Tỷ lệ điều kiện thời tiết mặt đường ướt (`mu = 0.5` với xác suất 30% ở thị trấn B)**:
   - *Dữ liệu thay thế*: Dữ liệu trạm khí tượng thủy văn địa phương về số giờ mưa trong năm và hệ số ma sát động bề mặt đường đo bằng xe chuyên dụng.
   - *Hướng sai lệch*: Nếu khu vực thực tế có mùa mưa kéo dài hoặc mặt đường dầu mỡ ($\mu < 0.4$), quãng đường phanh thực tế dài hơn đáng kể, khiến tỷ lệ va chạm thực tế cao hơn con số báo cáo.

---

## 8. Giới hạn

Ba điều mà xác suất báo cáo **không** phản ánh:
1. *Môi trường hình học giản lược*: Thế giới chỉ có một đoạn phố thẳng đơn tuyến, không có ngã tư phức tạp, bùng binh, xe máy luồn lách hay đèn tín hiệu giao thông tương tác.
2. *Mô hình tác nhân cơ học*: Các phương tiện xung quanh tuân thủ IDM cứng nhắc, không có phản xạ đánh lái khẩn cấp (evasive steering) hay các hành vi giận dữ, bất cẩn bất thường của con người.
3. *Cảm biến nhận thức hoàn hảo*: Hệ thống giả định cảm biến nhận biết chính xác $100\%$ vị trí và vận tốc của mọi vật thể trong tầm quét $120\text{ m}$ (trừ góc khuất hình học), bỏ qua độ trễ tính toán (latency), nhiễu tín hiệu và lỗi phát hiện/bỏ sót của thuật toán thị giác.

*Nếu được thêm một tuần*: Sẽ bổ sung mô hình hóa độ trễ nhận thức (perception latency $100 - 300\text{ ms}$) và thử nghiệm khả năng đánh lái chuyển làn khẩn cấp để đánh giá tính khả thi khi phanh đơn thuần không đủ cứu vãn va chạm.

---

## 9. Rắc rối của bạn (bước G + chương 10)

- **Rắc rối**: Thiết lập tình huống người đi bộ băng ngang đường tự phát (jaywalking) giữa phố từ vỉa hè phía Nam (băng qua làn E2 vào làn E1 của xe ta). Ngoài đời tương ứng với tình huống người đi bộ bất ngờ bước xuống lòng đường đón xe bus hoặc sang đường tùy tiện giữa phố.
- **Căn cứ**: Planner chỉ có logic giảm tốc thận trọng (`cautionRange`, `cautionV`) đối với vạch qua đường cố định `WALK_X = 122m`. Với người đi bộ qua đường ở vị trí bất kỳ khác, planner chỉ coi là vật cản khi đã nằm hẳn trong làn xe ta ($|lat| \le 1.9\text{ m}$), khiến xe ta không kịp phản xạ khi đang di chuyển ở tốc độ hành trình $12.5\text{ m/s}$.
- **Giả định (`hazard.json: why`)**:
  - `space`: Vị trí `h0_x`: $[190, 230]\text{ m}$; tốc độ đi bộ `h0_v`: $[1.2, 1.8]\text{ m/s}$ (theo tiêu chuẩn thiết kế ITE/MUTCD cho người trưởng thành); cự ly kích hoạt `h0_trig`: $[20, 32]\text{ m}$.
  - `exposure`: $0.03$ lần/chuyến đi (dựa trên tần suất xung đột người đi bộ giữa phố trong các nghiên cứu an toàn giao thông đô thị của FHWA/NHTSA).
- **Kết quả (trích xuất từ `evidence/mine.json`)**:
  - Tỷ lệ kịch bản khả thi: `accepted = 0.9953` ($99.53\%$ hợp lệ, `infeasible = 7` trên $1500$ mẫu).
  - Policy gốc: `avoidable = 0`, `unavoidable = 0`, `unattributed = 0`, `low_speed = 0`.
  - Policy được chọn (`cutinLat = 3.3`): `avoidable = 0`, `unavoidable = 0`, `unattributed = 0`, `low_speed = 0`.
  - Xác suất lỗi: $P(\text{lỗi planner} \mid \text{rắc rối}) = 0.00$, khoảng Wilson 95%: $[0.0, 2.57 \times 10^{-3}]$.
  - Tần suất mỗi chuyến đi: $0.00$ (cận trên $7.70 \times 10^{-5}$/chuyến đi). So với tần suất lách xe buýt (`bus_bypass` $\approx 1.33 \times 10^{-3}$), nguy cơ này thấp hơn do cơ chế người đi bộ tự dừng khi thấy thân xe phía trước (`PED_LOOK`).
- **Hình 3D của rắc rối tự dựng**: `evidence/shots/ch10-studio.png` cho thấy H0 người đi bộ băng ngang từ phía nam, `x = 210 m`, `v = 1.50 m/s`, kích hoạt ở `26 m`, khớp với `hazard.json`. Trong lần chạy đang xem, planner và planner gốc đều không va chạm, phù hợp với kết quả trong `mine.json`.
- **Trải nghiệm tự lái**: Khi tự điều khiển trong Chapter 10 Studio, người lái quan sát thấy người đi bộ từ lúc họ rời lề đường và chủ động rà phanh hạ tốc độ từ xa xuống $8\text{ m/s}$, giúp xe đi qua an toàn và êm ái hơn việc planner giữ nguyên tốc độ rồi mới phanh gấp khi người đi bộ tới gần tim đường.
- **Vì sao "tránh được" là chuẩn lạc quan (hai chiều)**:
  1. *Lạc quan theo chiều biết trước*: Bộ kiểm tra giả định hệ thống phanh tức thời với gia tốc cực đại $\mu g$ ngay thời điểm nhìn thấy hiểm họa mà không có độ trễ nhận thức/phản xạ của con người ($\approx 1\text{ s}$), nên số va chạm "tránh được" tính toán nhiều hơn số vụ một planner thực tế có thể tránh.
  2. *Lạc quan theo chiều giới hạn hành động*: Bộ kiểm tra chỉ thử duy nhất phương án phanh thẳng dừng lại mà không thử nghiệm cơ động đánh lái tránh (evasive swerve), khiến một số tình huống có thể thoát hiểm bằng đánh lái bị bỏ qua.
- **Cách sửa**: Các núm chỉnh IDM hiện tại không thể khắc phục vì thiếu biến trạng thái không gian cho người đi bộ ngoài vạch kẻ. Cần bổ sung logic nhận diện sớm và dự báo giao cắt quỹ đạo (Time-to-Corridor) cho người đi bộ ở mọi vị trí trên vỉa hè để chủ động hạ tốc độ phòng ngừa từ khoảng cách xa.
