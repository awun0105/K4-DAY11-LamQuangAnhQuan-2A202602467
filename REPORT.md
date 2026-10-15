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

Bảng: với từng kiểu và từng seed trong `estimate.json`, ghi `p`, `hits`, `ess`, `top_share`. Ba seed có đồng ý với nhau không, và có đồng ý với Monte Carlo không? Ở đâu `top_share` lớn (> 0,5), và vì sao khi đó con số thực ra chỉ là trọng số của một mẫu? So ngân sách (`budget`) với số lần chạy Monte Carlo cần để có cùng độ chính xác. Phần `other` trong tổng lấy từ đâu, và vì sao không săn nó?

## 5. Các kiểu thất bại (bước C2 + 3D)

Mỗi kiểu bạn tìm thấy: một ảnh chụp trình xem 3D (chương 9) ở đúng lúc va chạm, và một câu mô tả chuỗi sự kiện (ai thấy gì, ai chờ gì). Kiểu nào cuộc săn **không** tìm thấy, và bạn tin điều đó ở mức nào? Nếu chỉ chạy **một** cuộc săn chung cho mọi kiểu thì điều gì xảy ra (gợi ý: `hunt.json: found` của từng cuộc săn)?

## 6. Sửa planner (bước E + 3D)

`fix.json`: policy được chọn, `fail` và `fail_hi` trên TRAIN, trên TEST (`fail_hi_seen` và `fail_hi`), và của policy gốc trên TEST; vì sao `fail_hi` của bản sửa có thể lớn hơn `fail` của policy gốc dù bản sửa tốt hơn thật? quãng đường và số lần phanh gấp. Một ảnh chụp 3D có **bóng mờ** (policy gốc) cho một kịch bản mà bản sửa tránh được. Có núm chỉnh nào "nghe thì an toàn hơn" nhưng thực tế làm tăng thất bại? Giải thích bằng cách thế giới phản ứng lại.

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
