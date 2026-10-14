MSSV: 202602631

# Lab 11 · Scenario Forge — báo cáo

Mỗi câu: trả lời ngắn, **trích số từ `evidence/`** (ghi tên file và trường, ví dụ `mc.json: crashes = 4`). Grader chạy lại với MSSV của bạn; số không khớp `evidence/` thì mất điểm phần D (RUBRIC), và câu dựa trên số sai khó được điểm đầy đủ. Không cần văn hay: một bảng nhỏ và hai, ba câu giải thích là đủ.

Thị trấn của bạn (`meta.json: _run.town`): **C**.

## 1. Phát lại log có trung thực không? (bước A)

Trong `replay.json`: bao nhiêu kịch bản bị bỏ vì người lái gây va chạm, bao nhiêu va chạm khi planner chạy trên log phát lại, bao nhiêu trong thế giới phản ứng, theo từng loại tiếp xúc. Giải thích vì sao phần lớn va chạm của "phát lại" là `rear`, và vì sao đó là va chạm **giả**. Chọn một kịch bản trong `replay_only` và nói chuyện gì đã xảy ra.

`replay.json: n = 400`, `dropped_human_crash_logs = 3`, nên còn `n_used = 397`. Planner trên log phát lại có `replay.crashes = 122`: `replay.contact.rear = 106`, `side = 16`, `front = 0`. Trong thế giới phản ứng, `reactive.crashes = 0` và `crash_in_both = 0` trên đúng tập này. 106/122 va chạm phát lại là từ phía sau. Xe bám đuôi trong log lặp nguyên quỹ đạo đã ghi khi đi sau người lái thật; nó không thể phanh theo planner khi planner giảm tốc sớm hơn. Vì thế va chạm với xe phía sau trong phép phát lại không chứng minh một tai nạn trong thế giới tương tác.

Ví dụ `replay.json: replay_only[0]` là kịch bản 0, `contact = rear`, còn `min_ttc_reactive = 2,37 s`. Planner thay đổi tốc độ so với người lái của log; xe bám đuôi theo quỹ đạo cũ chạm đuôi ego trong phát lại. Khi chạy phản ứng, xe bám đuôi thấy ego giảm tốc và điều chỉnh theo, nên cảnh ấy không có va chạm.

## 2. Kịch bản bất khả thi (bước B)

Liệt kê các luật trong `is_feasible` và căn cứ vật lý của từng luật (ma sát, gia tốc, tốc độ người). Trong `hunt.json`, cuộc săn nào sinh nhiều kịch bản bất khả thi nhất, và vì sao CEM lại bị kéo về phía đó? Điều gì xảy ra với ước lượng nếu bạn **đếm** chúng là va chạm?

`lab11/feasible.py: is_feasible` loại khi: (1) chuyển làn 3,5 m trong `cutDur` cần gia tốc ngang đỉnh `(π/cutDur)² × 3,5/2 > μg`, tức lố độ bám lốp; (2) `cutBrake > μg`, tức đòi phanh mạnh hơn ma sát cho phép; (3) `pedV > 2,5 m/s`, đã thành chạy thay vì người đi bộ; hoặc (4) `cutV` nằm ngoài `(0, 22] m/s`, tốc độ nhập làn không hợp lệ. Thời gian chuyển làn không dương cũng bị loại vì không định nghĩa được gia tốc.

`hunt.json: crosswalk_block.infeasible = 13` là nhiều nhất, so với `rear_ended = 9`, `cut_in = 4`, `bus_bypass = 3`. Kiểm tra các hàng `cloud.json: out = infeasible` cho thấy 29/29 vi phạm giới hạn gia tốc ngang; CEM dễ ưu tiên chuyển làn gấp trên đường ướt vì điểm nguy hiểm cao. Nếu cho các kịch bản không thể xảy ra ấy chạy hoặc đếm là va chạm, cuộc săn sẽ co vào miền phi vật lý và xác suất thất bại báo ra bị thổi phồng.

## 3. Monte Carlo cho bạn biết gì (bước C1)

`mc.json`: k, n, p̂ và khoảng Wilson 95 %. Với p̂ đó, cần bao nhiêu lần chạy để khoảng tin cậy hẹp lại còn ±20 % của p? Có kiểu thất bại nào **chưa hề xuất hiện** trong 3 000 lần chạy không? Con số đúng cần báo cho kiểu đó là gì?

`mc.json`: 5/3 000 va chạm, p̂ = 0,001667; Wilson 95 % = [0,000712; 0,003896]. Với p̂ này, xấp xỉ 96/p̂ = 57 600 lần chạy mới đạt sai số tương đối ±20 %. `cut_in`, `rear_ended` và `other` đều có 0 hit; không kết luận xác suất bằng 0. Cận trên Wilson 95 % của từng kiểu là 0,001279 (`mc.json: per_mode`).

## 4. Importance sampling, ESS và độ suy biến (bước C3)

Bảng: với từng kiểu và từng seed trong `estimate.json`, ghi `p`, `hits`, `ess`, `top_share`. Ba seed có đồng ý với nhau không, và có đồng ý với Monte Carlo không? Ở đâu `top_share` lớn (> 0,5), và vì sao khi đó con số thực ra chỉ là trọng số của một mẫu? So ngân sách (`budget`) với số lần chạy Monte Carlo cần để có cùng độ chính xác. Phần `other` trong tổng lấy từ đâu, và vì sao không săn nó?

| Kiểu (`estimate.json: seeds[].per`) | Seed | p | Hits | ESS | Top share |
| :-- | --: | --: | --: | --: | --: |
| `bus_bypass` | 0 | 0,001444 | 34 | 154,3 | 0,387 |
| `cut_in` | 0 | 0 | 0 | 118,3 | 0 |
| `crosswalk_block` | 0 | 0,00000856 | 84 | 130,0 | 0,347 |
| `rear_ended` | 0 | 0 | 0 | 118,0 | 0 |
| `bus_bypass` | 1 | 0,0006803 | 23 | 121,5 | **0,540** |
| `cut_in` | 1 | 0 | 0 | 142,2 | 0 |
| `crosswalk_block` | 1 | 0,00000371 | 74 | 121,6 | 0,452 |
| `rear_ended` | 1 | 0 | 0 | 113,0 | 0 |
| `bus_bypass` | 2 | 0,001027 | 31 | 137,3 | 0,392 |
| `cut_in` | 2 | 0 | 0 | 126,8 | 0 |
| `crosswalk_block` | 2 | 0,00000296 | 80 | 112,4 | 0,265 |
| `rear_ended` | 2 | 0 | 0 | 118,5 | 0 |

Tổng của ba seed là 0,001453; 0,000684; 0,001030 (`estimate.json: seeds[].total`): khác nhau hơn 2 lần nhưng đều nằm trong khoảng Wilson của MC. MC cho 0,001667, nên chưa thể coi các ước lượng IS đã ổn định. Ở seed 1, một mẫu `bus_bypass` mang 54,0 % tổng trọng số va chạm của kiểu đó; ước lượng nhạy mạnh với đúng mẫu ấy, dù có 23 hit. CEM tập trung vào miền nguy hiểm tìm được, nên `top_share` thấp ở seed khác cũng không chứng minh q phủ đủ mọi cách va chạm.

`estimate.json: budget = 16 800`, so với khoảng 57 600 lượt MC theo phép tính trên; ngân sách ít hơn khoảng 3,4 lần, nhưng ba seed chưa cho thấy độ chính xác ±20 %. `cut_in` và `rear_ended` có 0 hit ở cả ba seed: `upper_zero_hits = 3/(0,2×600) = 0,025` cho mỗi kiểu, không dùng 3/ESS và không báo xác suất chắc chắn bằng 0. `other` lấy từ `mc.json` vì không có cuộc săn riêng: 0/3 000, cận trên Wilson 0,001279 (`estimate.json: other`).

## 5. Các kiểu thất bại (bước C2 + 3D)

Mỗi kiểu bạn tìm thấy: một ảnh chụp trình xem 3D (chương 9) ở đúng lúc va chạm, và một câu mô tả chuỗi sự kiện (ai thấy gì, ai chờ gì). Kiểu nào cuộc săn **không** tìm thấy, và bạn tin điều đó ở mức nào? Nếu chỉ chạy **một** cuộc săn chung cho mọi kiểu thì điều gì xảy ra (gợi ý: `hunt.json: found` của từng cuộc săn)?

| Kiểu | Ảnh chương 9 | Chuỗi sự kiện |
| :-- | :-- | :-- |
| `bus_bypass` | [B-01, 12,7 s](evidence/shots/mssv-202602631-bus-bypass.png) | C2 tìm khe đủ rộng để chuyển làn bên xe buýt (`viewer.json: B-01.cutGap = 5,36 m`), rồi chậm lại khi lách qua. Xe ta tiến tới và chạm C2 lúc 12,65 s. Người đi bộ ở gần vạch nhưng không phải bên va chạm. |
| `crosswalk_block` | [C-01, 20,6 s](evidence/shots/mssv-202602631-crosswalk-block.png) | P1 chờ cho các xe **nhìn thấy** đủ xa trước khi qua đường; `pedLook = 0` nên không ngó qua xe buýt. Xe ta dừng với thân xe trên vạch, buộc P1 dừng ngay trước thân xe lúc 20,55 s. Đây là `block`, không phải tiếp xúc vật lý. |

`hunt.json: found` ghi 139 `bus_bypass` trong cuộc săn cùng tên và 62 `crosswalk_block` trong cuộc săn cùng tên. Cuộc săn `cut_in` và `rear_ended` không tìm được va chạm đúng kiểu; thay vào đó chúng gặp `bus_bypass` (lần lượt 30 và 13), riêng cuộc săn `rear_ended` còn gặp 5 `crosswalk_block`. Không có hit không chứng minh kiểu ấy bất khả thi: `estimate.json` vẫn ghi `upper_zero_hits = 0,025` cho mỗi kiểu ở từng seed. Một cuộc săn chung dễ dồn mẫu vào kiểu tìm được trước, đặc biệt `bus_bypass`, và bỏ sót kiểu hiếm khác.

## 6. Sửa planner (bước E + 3D)

`fix.json`: policy được chọn, `fail` và `fail_hi` trên TRAIN, trên TEST (`fail_hi_seen` và `fail_hi`), và của policy gốc trên TEST; vì sao `fail_hi` của bản sửa có thể lớn hơn `fail` của policy gốc dù bản sửa tốt hơn thật? quãng đường và số lần phanh gấp. Một ảnh chụp 3D có **bóng mờ** (policy gốc) cho một kịch bản mà bản sửa tránh được. Có núm chỉnh nào "nghe thì an toàn hơn" nhưng thực tế làm tăng thất bại? Giải thích bằng cách thế giới phản ứng lại.

`fix.json: chosen` chọn `cutinLat = 3,0`, `cautionV = 7,0`. Trên TRAIN, `fail = 3,10×10⁻⁶`, `fail_hi = 5,03×10⁻⁶`, progress = 204,6 m; baseline có `fail = 7,43×10⁻⁴`, `fail_hi = 1,18×10⁻³`, progress = 210,8 m. Bản sửa giữ khoảng 97,1% quãng đường TRAIN, vượt ngưỡng 95%.

| TEST (`fix.json`) | Fail | `fail_hi_seen` | `fail_hi` hợp lệ | Progress | Phanh gấp trung bình |
| :-- | --: | --: | --: | --: | --: |
| Policy chọn | 3,45×10⁻⁶ | 6,52×10⁻⁶ | 3,88×10⁻² | 204,0 m | 9,29 |
| Baseline | 1,69×10⁻³ | 3,07×10⁻³ | 4,35×10⁻³ | 211,1 m | 8,59 |

Policy chọn có `bus_bypass` trong `unseen`: không có hit không có nghĩa xác suất bằng 0. `fail_hi` cộng cận zero-hit 0,0375 của kiểu này và cận của va chạm khác trên 3.000 lượt tự nhiên, nên cao hơn `fail` của baseline. `fail_hi_seen` chỉ tóm tắt kiểu đã thấy; nó không phải cận đầy đủ. Ở TEST, bản sửa đi ít hơn 7,1 m và trung bình phanh gấp nhiều hơn khoảng 0,7 khung mỗi mẫu.

Ảnh [E-01, policy đã chọn tránh va chạm, bóng mờ baseline chạm lúc 12,5 s](evidence/shots/mssv-202602631-policy-comparison.png) dùng kịch bản `bus_bypass` từ `viewer.json: cloud row 1390`. Bản sửa không va chạm; baseline chạm C2. Đây là một ví dụ cụ thể, không chứng nhận rằng policy mới tránh mọi biến thể của kiểu này.

Núm nghe có vẻ an toàn hơn là tăng IDM `T` từ 1,6 lên 2,0 s để giữ khoảng cách lớn hơn. Nhưng hàng `T = 2,0` trong bảng TRAIN có `fail = 2,84×10⁻³`, `fail_hi = 5,17×10⁻³`, cao hơn baseline (`fail_hi = 1,18×10⁻³`). Ego giảm tốc và mở khe phía sau; C2 kiểm tra khe theo vị trí và vận tốc hiện thời nên có thể chấp nhận chuyển làn ở thời điểm khác. Xe khác cũng tiếp tục phản ứng với ego đã chậm hơn. Vì vậy thay đổi một núm không chỉ đổi khoảng cách của ego: nó đổi thời điểm và hành vi của các xe xung quanh, và số va chạm có thể tăng.

## 7. Từ mô phỏng ra đường thật

Phân phối tự nhiên trong `space.py` là giả định. Chọn **ba** giả định ảnh hưởng nhiều nhất đến con số của bạn (ví dụ khe chấp nhận của xe chuyển làn, tỉ lệ người đi bộ ngó qua xe buýt, ma sát đường ướt). Với mỗi cái: dữ liệu thật nào sẽ thay nó, lấy từ đâu, và nếu nó sai thì con số của bạn sai theo hướng nào?

| Giả định của thị trấn C (`avsim/space.py: VARIANTS`, `BIN`) | Dữ liệu thật để thay | Nếu giả định sai |
| :-- | :-- | :-- |
| Trung vị khe xe C2 chấp nhận là `cut_gap_median = 10 m`. | Trích quỹ đạo xe, tốc độ và khe trước/sau tại từng lần nhập làn gần xe buýt từ camera hoặc LiDAR bên đường; ước lượng phân phối khe được chấp nhận và cả khe bị từ chối. | Nếu lái xe thật chấp nhận khe nhỏ hơn 10 m, `mc.json: per_mode.bus_bypass.k = 4` có thể đang đánh giá thấp nguy cơ lách xe buýt. Khe lớn hơn có thể đổi chiều sai lệch; tương tác có thể khiến quan hệ không đơn điệu. |
| Xác suất P1 ngó qua xe buýt là `p_bin[pedLook] = 0,80`. | Ghi hình các lần người đi bộ đứng sát xe buýt, chú giải họ có dịch vị trí/ngó để thấy xe đến hay không; đồng bộ với góc nhìn thực tế bị che. | Nếu tỷ lệ ngó thật thấp hơn 80%, xe và người sẽ thiếu thông tin về nhau thường hơn; thất bại gần vạch và số `crosswalk_block` có thể bị đánh giá thấp. |
| Xác suất đường ướt `p_bin[wet] = 0,20`; ma sát mô phỏng `μ = 0,5` khi ướt, `0,8` khi khô (`avsim/space.py: to_params`). | Ghép log hành trình với mưa, độ ướt mặt đường và phép đo/ước lượng độ bám từ phanh ABS hoặc xe đo mặt đường, theo mùa và tuyến. | Nếu đường ướt thường hơn 20% hoặc độ bám ướt thấp hơn 0,5, khoảng dừng thực dài hơn và xác suất va chạm `mc.json: p = 0,001667` có thể bị đánh giá thấp. |

Ba hướng trên là dự đoán có điều kiện, không phải phép hiệu chỉnh đã đo: các tác nhân trong thế giới phản ứng lại thay đổi của nhau, nên cần chạy lại mô phỏng với phân phối được hiệu chỉnh rồi đối chiếu dữ liệu giữ riêng.

## 8. Giới hạn

Ba điều mà xác suất bạn báo **không** nói, ví dụ: thế giới chỉ có một con phố; người lái chỉ là IDM; cảm biến hoàn hảo trừ phần bị xe buýt che. Nếu được thêm một tuần, bạn sẽ đo gì tiếp?

1. `mc.json: p = 0,001667` là trên phố và 24 giây của mô phỏng (`avsim/core.py: T_END`), với một xe buýt và các tác nhân đã định sẵn. Nó không phải xác suất cho mọi chuyến đi, nút giao, thời tiết và mật độ giao thông của thành phố.
2. Xe khác dùng IDM và quy tắc chấp nhận khe cố định (`avsim/core.py`); họ không thể mất tập trung, bực bội, đổi ý hay thương lượng bằng tín hiệu. Các va chạm do những hành vi đó nằm ngoài `replay.json`, `mc.json` và `estimate.json`.
3. Cảm biến của ego trong lõi gần như hoàn hảo trừ che khuất đã mô hình hoá; không có bỏ sót phát hiện, nhận dạng nhầm hay độ trễ xử lý. Vì vậy khoảng tránh được bằng phanh trong `mine.json: avoidable` cũng không chứng nhận khả năng tránh trong hệ thống cảm biến thật.

Nếu có thêm một tuần, tôi sẽ thu thập dữ liệu đã đồng bộ camera/LiDAR, phanh và quỹ đạo trên nhiều đoạn phố có xe buýt, người băng ngang, cả điều kiện khô/ướt; ước lượng lại ba phân phối ở câu 7 và kiểm tra trên các chuyến đi giữ riêng, đặc biệt những va chạm xuất hiện sau khi thay policy.

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

**Tình huống và cơ chế.** H0 là người đi bộ từ lề Nam băng ngang tại `x = 189,14 m`, cách xa vạch `x = 122 m`; họ bắt đầu khi xe ta còn khoảng `48,49 m`. Cảnh mẫu có đường ướt (`μ = 0,5`). Planner chỉ dự đoán P1 ở vạch `x = 122 m`; với H0 ngoài vạch, nó chờ đến khi người đã ở gần làn xe mới phản ứng. Cảnh cụ thể trong `hazard.json` cho va chạm ego–H0 lúc `22,50 s` với policy gốc và `22,85 s` với policy đã chọn. Đây là va chạm có thể tránh bằng phanh sớm, và bản sửa ở bước 6 vẫn chưa giải quyết được.

**Căn cứ cho `space` và `exposure`.** Ba khoảng là `h0_x = [180, 220] m`, `h0_trig = [44, 56] m`, `h0_v = [1,3, 1,8] m/s`. Khoảng vị trí rộng 40 m quanh đoạn phố ngoài vạch, thay vì dịch người đến đúng vị trí dễ va chạm nhất. Khoảng kích hoạt tương đương khoảng 3,5–4,5 s ở tốc độ danh định 12,5 m/s, cùng cỡ thời gian phản ứng trong [nghiên cứu tương tác xe–người đi bộ của Sun và cộng sự](https://pureadmin.qub.ac.uk/ws/portalfiles/portal/388245508/sensors-22-07860.pdf), bảng 1. Khoảng tốc độ nằm trong dải đi bộ 0,8–1,8 m/s của [FHWA](https://www.fhwa.dot.gov/publications/research/safety/04091/02.cfm). Nghiên cứu Sun sàng lọc 98 lượt băng ngang của một người, ngoài nút giao, trong khoảng 980 km lái xe ở khuôn viên; dùng như **proxy**, một lượt đi 0,2 km trên phố mô phỏng có `exposure ≈ (98/980) × 0,2 = 0,02` lần/lượt. Đây chưa phải tần suất đã đo tại thị trấn C hoặc riêng hành vi băng ngang bất ngờ; cần quan sát tại địa điểm và đo chiều dài chuyến đi thực để hiệu chỉnh. `hazard.json: why` lưu công thức và nguồn.

`mine.json` chạy **đầy đủ 1.500** biến thể trên giao thông thị trấn C, giữ 1.491 biến thể khả thi (`accepted = 0,994`, loại 9). Trong bảng, `low_speed` là cờ nằm trong các loại va chạm, không cộng thêm vào `crashes`.

| Policy | Va chạm | `avoidable` | `unavoidable` | `unattributed` | `low_speed` | P(lỗi planner \| có H0), Wilson 95% | Tần suất/lượt với exposure 0,02 |
| :-- | --: | --: | --: | --: | --: | :-- | --: |
| Gốc | 148 | 144 | 0 | 4 | 0 | 0,0966 [0,0826; 0,1126] | 0,001932 |
| Đã chọn | 96 | 96 | 0 | 0 | 1 | 0,0644 [0,0530; 0,0780] | 0,001288 |

`avoidable_foreknown` lần lượt cũng là 144 và 96; trong các vụ này, phép thử không thấy thêm vụ nào do che tầm nhìn. Để đặt cạnh kiểu lách xe buýt của policy gốc, `mc.json` có 4/3.000 ≈ 0,00133 mỗi lượt. Ước lượng cho H0 sau nhân exposure là khoảng 0,00193 mỗi lượt, cùng bậc và lớn hơn khoảng 1,45 lần theo ước lượng điểm. Hai con số còn bất định, đặc biệt do proxy exposure và chỉ 4 vụ lách xe buýt trong MC.

**Ảnh và tự lái.** [Chương 9, mẫu GC-02 ở khung va chạm 22,85 s](evidence/shots/mssv-202602631-custom-hazard.png) cho thấy policy đã chọn vẫn chạm H0; bóng mờ là policy gốc chạm lúc 22,50 s. Trong [chương 10 ở chế độ “Bạn”](evidence/shots/mssv-202602631-manual-brake.png), tôi giữ tốc độ trước đó rồi giữ phanh từ `16 s`; bản ghi chân phanh nằm ở `evidence/manual-drive.json`. Lượt lái này không va chạm, trong khi planner gốc trên cảnh đó va H0. Người điều khiển đã xem trước cảnh và biết thời điểm, nên đây chỉ là minh họa cho khả năng tránh, không phải bằng chứng người lái bất kỳ sẽ phản ứng như vậy.

**Sửa planner và giới hạn phép thử.** Hai núm `cutinLat` và `cautionV` của bản sửa không xử lý người ngoài vạch. Planner cần phát hiện và dự đoán quỹ đạo **mọi** người đi bộ gần đường, ước lượng thời điểm họ vào làn xe theo vị trí băng ngang của từng người, rồi giảm tốc hoặc phanh khẩn cấp khi có nguy cơ giao cắt; không thể chỉ tham chiếu P1 tại vạch. Chuẩn `avoidable` lạc quan vì phanh ngay khung đầu tiên rắc rối bắt đầu và được thấy, không tính độ trễ nhận biết hay thao tác; đồng thời nó chỉ thử phanh, bỏ qua cơ hội tránh bằng đánh lái, nên có thể đếm thiếu những vụ thực ra tránh được.
