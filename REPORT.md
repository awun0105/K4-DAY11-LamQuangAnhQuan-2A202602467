MSSV: 202602467

# Lab 11 · Scenario Forge — báo cáo

Mỗi câu: trả lời ngắn, **trích số từ `evidence/`** (ghi tên file và trường, ví dụ `mc.json: crashes = 4`). Grader chạy lại với MSSV của bạn; số không khớp `evidence/` thì mất điểm phần D (RUBRIC), và câu dựa trên số sai khó được điểm đầy đủ. Không cần văn hay: một bảng nhỏ và hai, ba câu giải thích là đủ.

Thị trấn của bạn (`meta.json: _run.town`): **A**

## 1. Phát lại log có trung thực không? (bước A)

`replay.json`: `n` = 400, `dropped_human_crash_logs` = 1 (bỏ vì chính người lái va chạm), `n_used` = 399.

| | va chạm | rear | side |
|---|---|---|---|
| phát lại (`replay`) | 124 | 113 | 11 |
| phản ứng (`reactive`) | 1 | 0 | 1 |

`crash_in_both` = 1, `replay_only` có 20 kịch bản (ví dụ #12, #14, #15: đều `rear`).

Vì sao phần lớn là `rear` và là va chạm **giả**: trong log, xe phía sau (TG) lái đúng như đã ghi với người lái thật. Planner của ta phanh sớm hơn người lái nhưng log không biết điều đó, nên xe sau vẫn không phanh theo và tông vào đuôi ta. Trong thế giới phản ứng, xe sau là IDM nên nhìn thấy ta phanh và giãn khoảng cách. Ví dụ kịch bản #12 (`replay_only`): va chạm `rear` ở log phát lại, còn trong thế giới phản ứng TTC nhỏ nhất vẫn là 2,61 s (`min_ttc_reactive`), tức không có nguy cơ thật.

## 2. Kịch bản bất khả thi (bước B)

Bốn luật trong `is_feasible` (`lab11/feasible.py`):
1. Đổi làn cần a_lat = (π/cutDur)²·3,5/2 > μg: vượt độ bám của lốp thì không xe nào làm được.
2. `cutBrake` > μg: không phanh mạnh hơn ma sát cho phép.
3. `pedV` > 2,5 m/s: người đi bộ không chạy nhanh vậy khi băng đường.
4. `cutV` ngoài (0, 22] m/s: xe đứng yên hoặc nhanh quá tốc độ của phố này.

Số kịch bản bất khả thi bị bỏ (`hunt.json: <kiểu>.infeasible`): `bus_bypass` 2, `cut_in` 4, `rear_ended` 4, **`crosswalk_block` 17** (nhiều nhất). CEM bị kéo về phía đó vì các giá trị cực đoan (người đi bộ chạy nhanh, đổi làn rất gấp) làm điểm nguy hiểm cao nhất. Trong `mc.json` có `infeasible` = 12 trên 3 000 lần rút tự nhiên. Nếu **đếm** chúng là va chạm thì ước lượng bị thổi phồng bằng những tình huống không thể xảy ra, và hunt "tìm thấy" lỗi không có thật.

## 3. Monte Carlo cho bạn biết gì (bước C1)

`mc.json`: k = 7, n = 3 000, p̂ = 2,33e-3, Wilson 95 % = [1,13e-3 ; 4,81e-3] (cả 7 là `bus_bypass`).

Để khoảng tin cậy hẹp còn ±20 % của p: n ≈ 1,96²/0,2² / p ≈ 96/p ≈ 96 / 2,33e-3 ≈ **41 000 lần chạy**, gấp ~14 lần ngân sách hiện tại.

Các kiểu chưa từng xuất hiện trong 3 000 lần: `cut_in`, `crosswalk_block`, `rear_ended`, `other` (đều k = 0). Con số đúng không phải 0 mà là cận trên Wilson 95 %: `mc.json: per_mode.<kiểu>.hi` = 1,28e-3.

## 4. Importance sampling, ESS và độ suy biến (bước C3)

`estimate.json` (`budget` = 16 800 lần chạy mô phỏng; n = 600 mẫu mỗi kiểu mỗi seed):

| seed | kiểu | p | hits | ess | top_share |
|---|---|---|---|---|---|
| 0 | bus_bypass | 3,74e-4 | 36 | 124 | 0,54 |
| 0 | crosswalk_block | 2,13e-7 | 96 | 137 | 0,36 |
| 1 | bus_bypass | 6,97e-4 | 67 | 119 | 0,48 |
| 1 | crosswalk_block | 2,95e-7 | 94 | 102 | 0,42 |
| 2 | bus_bypass | 1,64e-3 | 56 | 125 | 0,38 |
| 2 | crosswalk_block | 1,43e-6 | 119 | 128 | 0,48 |

`cut_in` và `rear_ended` ở cả 3 seed: p = 0, hits = 0, `upper_zero_hits` = 0,025 (cận trên, không phải 0). `total` theo seed: 3,74e-4 / 6,97e-4 / 1,64e-3.

Ba seed **không đồng ý** (chênh ~4,4 lần) và đều **thấp hơn** Monte Carlo (2,33e-3), đúng với cảnh báo "IS thường thấp hơn 1,5–4 lần". Seed 2 gần MC nhất. `top_share` > 0,5 chỉ ở seed 0 `bus_bypass` (0,54): hơn một nửa con số là trọng số của đúng một mẫu, nên ước lượng thực chất là một điểm. Các mẫu khác cũng gần 0,5, nên con số kém tin cậy.

So ngân sách: IS dùng 16 800 lần chạy (3 seed); để Monte Carlo có độ chính xác ±20 % tại p ≈ 1e-3 cần ~96/1e-3 ≈ 96 000 lần, và với `crosswalk_block` (p ~ 1e-7 đến 1e-6) cần hàng trăm triệu lần: IS rẻ hơn nhiều bậc, nhưng con số kém chắc. Phần `other` lấy từ Monte Carlo (`estimate.json: other` = k 0, n 3 000, p = 0, hi = 1,28e-3, source `mc.json`) vì nó không thuộc kiểu nào nên CEM không săn; tổng = các kiểu + `other`.

## 5. Các kiểu thất bại (bước C2 + 3D)

Kết quả `hunt.json: <cuộc săn>.found`:

| cuộc săn | tìm thấy |
|---|---|
| bus_bypass | bus_bypass 95 |
| cut_in | bus_bypass 52 (không có cut_in) |
| crosswalk_block | crosswalk_block 67, bus_bypass 1 |
| rear_ended | crosswalk_block 7, bus_bypass 4 (không có rear_ended) |

- **bus_bypass**: xe chuyển làn C2 (chậm, x < 112 m) lách qua xe buýt đỗ; xe ta không thấy nó đủ sớm vì xe buýt che, rồi đâm vào.
- **crosswalk_block**: người đi bộ P1 bước ra từ sau xe buýt đỗ; xe ta phanh gấp nhưng dừng khi thân xe còn nằm trên vạch, chặn lối người đi bộ.
- **cut_in và rear_ended**: cuộc săn **không tìm thấy**; chỉ là kết quả "chưa thấy", không phải bằng chứng không tồn tại. Cận trên 95 % là 2,5e-2 (`upper_zero_hits`) ở IS và 1,28e-3 ở Monte Carlo.

Nếu chỉ chạy một cuộc săn chung, CEM co về kiểu dễ nhất (`bus_bypass`, nó chiếm mặt `found` của cả `cut_in` lẫn `rear_ended`), và các kiểu hiếm như `crosswalk_block` sẽ không được khám phá.

Ảnh 3D (chương 9): *chưa chụp trong lần chạy này*; `evidence/viewer.json` đã export sẵn để mở trong `viewer/index.html`.

## 6. Sửa planner (bước E + 3D)

`fix.json`: policy được chọn bằng `choose_policy` trên TRAIN: `{"cutinLat": 3.0}`.

| | fail | fail_hi_seen | fail_hi (hợp lệ) | quãng đường (m) | phanh gấp |
|---|---|---|---|---|---|
| bản sửa, TRAIN (`chosen`) | 1,10e-7 | 2,11e-7 | 2,11e-7 | 212,5 | 7,94 |
| bản sửa, TEST (`chosen_test`) | 2,77e-8 | 4,54e-8 | **3,88e-2** | 214,6 | 7,80 |
| policy gốc, TEST (`baseline_test`) | 1,53e-3 | 3,69e-3 | 4,97e-3 | 214,6 | 8,47 |

Vì sao `fail_hi` của bản sửa (3,88e-2) lớn hơn `fail` của bản gốc (1,53e-3) dù bản sửa tốt hơn: kiểu `bus_bypass` được đánh dấu `unseen` (0 va chạm trên TEST), nên nó vẫn cộng cận trên 3/(0,2·n) ≈ 3,75e-2 vào `fail_hi`. Với n = 400, không thể chứng nhận được một kiểu đã biến mất. Bản sửa **có vẻ** tốt hơn (40 000 lần nhỏ hơn ở `fail_hi_seen`), nhưng dữ liệu không đủ để chứng nhận.

Nhắc thêm: `choose_policy` chọn bản này vì mọi ứng viên có `cutinLat` ≥ 3,0 đều có TRAIN `fail_hi` = 0 do `unseen`: đúng cái bẫy README cảnh báo. Con số nên tin là TEST, không phải TRAIN.

Núm "nghe an toàn hơn" nhưng tệ hơn: `T` = 2,0 (giãn khoảng cách theo sau) có `fail_hi` TRAIN = 6,48e-4, cao hơn policy gốc (3,42e-4). Giải thích khả dĩ: xe chuyển làn chờ một khe đủ rộng (gap acceptance), nên khoảng trống lớn hơn trước mặt ta mời xe C2 chen vào sớm hơn và gần hơn. Đây là suy luận từ cơ chế, chưa tách riêng để kiểm chứng.

Ảnh 3D có bóng mờ: *chưa chụp trong lần chạy này*.

## 7. Từ mô phỏng ra đường thật

Phân phối tự nhiên trong `space.py` là giả định. Chọn **ba** giả định ảnh hưởng nhiều nhất đến con số của bạn (ví dụ khe chấp nhận của xe chuyển làn, tỉ lệ người đi bộ ngó qua xe buýt, ma sát đường ướt). Với mỗi cái: dữ liệu thật nào sẽ thay nó, lấy từ đâu, và nếu nó sai thì con số của bạn sai theo hướng nào?

## 8. Giới hạn

Ba điều mà xác suất bạn báo **không** nói, ví dụ: thế giới chỉ có một con phố; người lái chỉ là IDM; cảm biến hoàn hảo trừ phần bị xe buýt che. Nếu được thêm một tuần, bạn sẽ đo gì tiếp?

## 9. Rắc rối của bạn (bước G + chương 10)

- **Rắc rối:** bạn dựng gì, và ngoài đời nó giống tình huống nào.
- **Căn cứ:** vì sao bạn nghĩ planner sẽ xử lý kém.
- **Giả định:** `space` và `exposure` lấy từ đâu (`hazard.json: why`).
- **Kết quả**, theo `mine.json`:
  - số va chạm `avoidable` (tránh được bằng phanh hết cỡ, biết trước), `unavoidable`, `unattributed`, và `low_speed`, với policy gốc và với policy bạn chọn; `accepted` (bao nhiêu phần `space` của bạn là khả thi);
  - P(lỗi planner | rắc rối) kèm khoảng Wilson;
  - tần suất mỗi chuyến, đặt cạnh tần suất lách xe buýt của bạn.
- **Hình:** một ảnh chương 9 hoặc 10 ở lúc va chạm.
- **Bạn tự lái thì sao:** nếu bạn đã thử lái, bạn tránh được không, và bạn làm gì khác planner?
- **Cách sửa:** núm chỉnh nào sửa được rắc rối này? Nếu không núm nào sửa được, planner cần thêm **logic gì** (mô tả, không cần code)?
