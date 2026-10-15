# REPORT — Lab Day 12

> Điền **bằng lời và số liệu của chính bạn** (lấy từ màn hình và từ `evidence/*.json`). Mỗi câu 4–8 dòng là đủ. Số không có trong bằng chứng của bạn sẽ bị trừ điểm. Báo cáo công khai trên GitHub; giảng viên đối chiếu với `evidence/` và với bài của các bạn khác.

| | |
|---|---|
| Họ và tên | `Dinh Lenh Tien Anh` |
| MSSV (đúng bằng `--student-id` bạn đã chạy) | `2A202602928` |
| GitHub | `AIVIETNAM-AIO-AnhDinh` |
| Seed chia dữ liệu / seed mô hình (xem `evidence/closedloop.json`, mục `student`) | `6900 / 4544 (seed huấn luyện 4544..4548)` |
| Số seed huấn luyện bạn dùng (`--seeds`) | `5` |

## Kết quả cuối

Điền từ `evidence/closedloop.json`, `openloop.json`, `gate.json` (cho mỗi mô hình học, lấy **dải qua các seed**).

| Bộ điều khiển | L2 vòng hở (m) | Va chạm vòng kín trên log (%) | Quyết định cổng |
|---|---|---|---|
| constant-velocity | 1.253 | 20.26 (584/2883, CI95 [18.83, 21.76]) | BLOCK (rớt R2, R3, R4, R6) |
| IDM | 1.494 | 0.03 (1/2883, CI95 [0.01, 0.20]) | (baseline) |
| EgoMLP | 0.763 .. 0.768 (TB 0.766) | 15.26 .. 29.31 | BLOCK (rớt R2, R3, R4, R5, R6) |
| PercMLP | 0.625 .. 0.630 (TB 0.628) | 0.66 .. 2.01 | BLOCK (rớt R2, R3, R4, R5, R6) |
| PercMLP+ (sau khi sửa) | 0.819 ± 0.026 | 0.07 .. 0.21 | RELEASE (qua cả 7 quy tắc) |

## Câu 1 · Dữ liệu
Bước 0 cho thấy ba vấn đề. (1) Cột `Acc_LV` gần như không liên quan đến đạo hàm tốc độ xe dẫn: tương quan chỉ **0.045**, trong khi cột tương ứng của xe sau đạt **0.929**. Vì vậy tôi không dùng `Acc_LV` mà dùng gia tốc suy ra từ `Speed_LV` **đã làm mượt** (`a_lead`). Với gia tốc này, phân vị p0.1 của gia tốc xe dẫn là −2.77 m/s², và bước 2 khai thác được 614 ca phanh.
(2) Có **383 quỹ đạo trùng hệt nhau** (9993 → 9610 sau khi loại), nằm ở các ID cách xa nhau. Nếu chia ngẫu nhiên, cùng một quỹ đạo có thể rơi vào cả train lẫn test, khiến điểm test bị "rò" (mô hình đã thấy đáp án).
(3) Các cửa sổ (2 s lịch sử → 3 s tương lai, bước trượt 0.2 s) trong **cùng một quỹ đạo** chồng lên nhau và gần như giống hệt. Chia theo khung hình sẽ đặt các cửa sổ gần giống nhau vào cả hai tập. Vì vậy phải chia theo **quỹ đạo**: 2883 quỹ đạo giữ lại làm test, cho 72002 cửa sổ test so với 167131 cửa sổ train. Ngoài ra tôi đã kiểm tra `Spatial_Gap` = hiệu vị trí − 4.5 m chiều dài xe (sai lệch tối đa 3.6e-11 m), nên khoảng cách đáng tin.

## Câu 2 · Phát lại
Log có 600 đoạn (9.5 MB). Chạy lại cùng mã cho 0/600 đoạn khác, giống từng bit. Bản refactor float32 khác bit ở **600/600 đoạn** nhưng **0 đoạn vượt dung sai** (atol = rtol = 1e-3 m/s²; lệch lớn nhất 9.7e-5). Phép cộng số thực không có tính kết hợp ((0.1+0.2)+0.3 = 0.6000000000000001 ≠ 0.6), nên so từng bit là phép thử sai. Dung sai 1e-3 m/s² nhỏ hơn rất nhiều so với mức ảnh hưởng đến xe, nhưng vẫn lớn hơn ~10 lần nhiễu float32.
Thay đổi AEB làm lệch **8/600 đoạn** ([56, 76, 126, 181, 188, 229, 320, 458], lệch lớn nhất 1.84 m/s²). Khi mở đoạn 56, chỉ các khung 98–101 bị lệch, có TTC = 1.47; 1.43; 1.43; 1.49 s (dưới ngưỡng 1.5 s), và lệnh đổi từ khoảng −5.4..−5.9 xuống −6.0. Đoạn 76 lệch ở khung 73–79 với TTC 1.38–1.44 s. AEB chỉ kích hoạt khi TTC < 1.5 s, nên mọi khung khác giữ nguyên.
Phát nhanh nhất có 0 khung trễ và 0 khung rớt, vì nó chờ bộ điều khiển xử lý xong mới gửi thông điệp tiếp theo, nên hạn chót không tồn tại. Phát theo thời gian thật lộ ra **587 khung trễ + 583 khung rớt / 60270**, làm đổi lệnh ở **377/600 đoạn** (lệch tới 12.45 m/s²). Nguyên nhân là khung xử lý mất 140 ms, vượt chu kỳ 100 ms, nên khung tiếp theo bị bỏ và lệnh cũ bị giữ lại.

## Câu 3 · Kịch bản
Lưới cắt vào của tôi: tốc độ cruise **14 m/s**, khoảng cách 2.76–59.31 m (p2..p98 của log), tốc độ tiến gần 0–14 m/s, **60 × 45 = 2700 ô**. Theo biên vật lý (phanh 9 m/s² sau thời gian trễ), **3.6 %** ô không ai tránh được khi t_react = 0 s, tăng lên **10.2 %** khi t_react = 0.6 s. Vùng đâm của IDM cũng tăng 3.6 % → 10.2 %, **trùng khớp** biên bất khả thi (phần đâm do lỗi bộ điều khiển = 0.0 %). Kiểm chứng FullBrake cho 0 ô lệch ở cả hai độ trễ. Vậy mọi va chạm của IDM trên lưới đều do vật lý, không phải do IDM. Ngược lại, constant-velocity đâm ở 70.4 % lưới tuy tránh được (t_react 0.6 s), tức đó là lỗi của nó.
Log không phủ vùng khó. Người lái trong log phanh mạnh nhất (p99.9) chỉ **3.62 m/s²**, trong khi cơ cấu chấp hành làm được 9. **12.7 %** lưới đòi phanh mạnh hơn mức mọi người lái trong log từng phanh. 614 ca khai thác (6.4 % quỹ đạo, phanh đỉnh p50 −2.5, p99 −4.2 m/s²), tức các chấm trắng trên `scenarios_idm.png`, tập trung ở góc khoảng cách nhỏ, tốc độ tiến gần thấp. Vùng tiến gần nhanh gần đường biên hầu như trống, nên mô hình học từ log chưa từng thấy vùng này.

## Câu 4 · Vòng hở và vòng kín
| Bộ điều khiển | L2 vòng hở (m) | Hạng | Va chạm vòng kín (TB qua seed) | Hạng |
|---|---|---|---|---|
| constant-velocity | 1.253 | 3 | 20.3 % | 3 |
| IDM | 1.494 | 4 | 0.03 % | 1 |
| EgoMLP | 0.766 | 2 | 23.8 % | 4 |
| PercMLP | 0.628 | 1 | 1.2 % | 2 |

Bốn cặp đổi thứ tự: (const-vel, IDM), (const-vel, EgoMLP), (IDM, EgoMLP), (IDM, PercMLP). **IDM** có L2 tệ nhất (1.494 m) vì nó không bắt chước người: nó giữ thời khoảng cách và gia tốc theo luật riêng, nên quỹ đạo dự báo lệch khỏi người lái thật. Nhưng L2 chỉ đo "giống người bao nhiêu", không đo "an toàn không". Trong vòng kín, IDM dùng khoảng cách và tốc độ tiến gần làm phản hồi, sửa sai ở mỗi bước, nên chỉ đâm 1/2883 quỹ đạo, và 0/171 trên lát cắt xe dẫn phanh.
**EgoMLP** chỉ nhìn lịch sử tốc độ của chính nó. Trên log, lịch sử ấy là của *người lái*, người đã phản ứng với xe dẫn, nên quán tính đó dự báo tốt tương lai gần (L2 0.766 m, tốt hơn 1.253 m của const-vel). Khi tự lái, lịch sử là của chính nó và không còn chứa tín hiệu nào về xe dẫn. Nó thiếu khoảng cách và tốc độ xe dẫn, nên không "thấy" xe dẫn phanh: đâm 15.3–29.3 % quỹ đạo và 63.2 % trên lát cắt phanh (seed xấu nhất). L2 trung bình còn che lỗi vì lát cắt phanh chỉ chiếm 4.5 % số cửa sổ (L2 của EgoMLP trên lát cắt là 1.12 m so với 0.766 m trung bình).

## Câu 5 · Một lần chạy chưa đủ
Cùng dữ liệu và kiến trúc, chỉ khác seed huấn luyện, EgoMLP đâm **15.26 % → 29.31 %** qua 5 seed (dải ~14 điểm). Khoảng Wilson 95 % của seed tốt nhất (4547) là [14.0, 16.6] %, của seed xấu nhất (4544) là [27.7, 31.0] %. Hai khoảng **không chồng nhau**. Khoảng Wilson chỉ đo độ bất định do mẫu test hữu hạn (n = 2883), không đo được độ bất định do huấn luyện. Phương sai giữa các seed lớn hơn nhiều so với độ rộng khoảng tin cậy.
Nếu chỉ chạy một seed, tôi có thể báo "EgoMLP đâm 15.3 % ± 1.3" hoặc "29.3 % ± 1.7", và cả hai đều trông chắc chắn. Với PercMLP còn nguy hiểm hơn: seed 4548 có 0.66 %, cận trên CI 1.03 %, tức gần như lọt qua R2 (ngưỡng 0.03 % + 1 điểm = 1.03 %). Trong khi đó seed 4545 có 2.01 % (cận trên 2.59 %). Một lần chạy may mắn có thể đưa một mô hình kém qua cổng. Vì vậy cổng lấy **seed xấu nhất** cho R2 và có R5 yêu cầu dải ≤ 1 điểm (PercMLP rớt R5 với dải 0.7..2.0 %).

## Câu 6 · Tác nhân phản ứng
Người lái trong log giữ thời khoảng cách trung vị **1.87 s**. Tôi hiệu chỉnh xe sau IDM theo khoảng cách Wasserstein giữa phân bố thời khoảng cách mô phỏng và log, và chọn được **T = 1.4 s** (W1 = 0.347; T = 1.2 cho 0.349, T = 2.4 cho 0.88). Một xe không bao giờ phản ứng có W1 = 0.86, tức kém chân thực hơn ~2.5 lần.
Lưới cắt vào, t_react = 0.6 s, tỷ lệ ô có va chạm bất kỳ (trước hoặc sau):

| Bộ điều khiển | Xe sau phát lại | Xe sau phản ứng | Va chạm sau (phát lại / phản ứng) |
|---|---|---|---|
| const-vel | 80.6 % | 80.6 % | 0.0 / 0.0 % |
| IDM | 80.9 % | 10.2 % | 70.6 / 0.0 % |
| EgoMLP | 83.2 % | 83.2 % | 0.0 / 0.0 % |
| PercMLP | 73.9 % | 43.7 % | 30.2 / 0.0 % |
| PercMLP+ | 81.5 % | 11.2 % | 71.1 / 0.0 % |

(t_react = 0 s: IDM 80.3 % → 3.6 %, phía sau 76.7 % → 0.0 %.) Với xe sau phát lại, IDM bị tính **70.6 %** va chạm phía sau. Đây là **va chạm giả**: xe sau phát lại chạy theo quỹ đạo người trong log, không biết xe thử vừa phanh đúng (phanh vì có xe cắt vào), nên đâm vào. Một người lái thật (xe phản ứng) không đâm (0.0 %). Trong thế giới phát lại, IDM (80.9 %) trông **tệ hơn** const-vel (80.6 %), còn PercMLP (73.9 %) trông tốt nhất, vì bộ không phanh thì không bao giờ "bị" đâm từ phía sau. Khi xe sau phản ứng, thứ tự đảo hẳn: IDM 10.2 % (đúng bằng vùng bất khả thi) là tốt nhất, const-vel 80.6 % là tệ nhất. Kiểm thử với tác nhân không phản ứng sẽ phạt đúng hành vi an toàn.

## Câu 7 · Cổng phát hành và sửa
PercMLP bị chặn bởi 5 quy tắc: R2 (seed xấu nhất 2.01 %, cận trên 2.59 % > 0.03 % + 1), R3 (7.0 % trên 171 ca phanh, cận trên 11.9 % > 5), R4 (33.5 % lưới tránh được mà vẫn đâm, so với 0.0 % + 2), R5 (0.7..2.0 % qua seed), R6 (43.7 % so với 10.2 % + 2). Có hai nguyên nhân gốc:
(a) **Kịch bản log không chứa** (R4, R6): cắt vào với tốc độ tiến gần lớn nằm ngoài phân bố log. Người lái chỉ phanh tới 3.62 m/s², và 12.7 % lưới đòi phanh mạnh hơn. Ở tốc độ cruise 20 m/s chưa từng thấy, PercMLP đâm 30.5..49.1 % lưới.
(b) **Sai số tích luỹ** (R2, R3, R5): L2 vòng hở tốt nhất (0.628 m), nhưng khi tự lái, sai số nhỏ đưa nó vào các trạng thái người lái không bao giờ đi vào (gần hơn, nhanh hơn), nơi nó chưa học cách sửa sai.
Cách sửa: (1) thêm 197160 cửa sổ từ 20000 ca cắt vào mô phỏng, nhãn là hành động của IDM, chữa (a); (2) DAgger 2 vòng: gắn nhãn IDM tại chính các trạng thái mô hình tự đi vào (~894k cửa sổ), chữa (b). Sau sửa, PercMLP+ qua cả 7 quy tắc: R2 0.21 % (cận trên 0.45 %), R3 0.0 % (cận trên 2.2 %), R4 1.0 %, R5 0.1..0.2 %, R6 11.2 %, và ở 20 m/s chỉ còn 0.1..1.0 %. L2 vòng hở lại **tệ đi** (0.628 → 0.819 m) trong khi va chạm giảm ~10 lần. Điều này một lần nữa cho thấy điểm vòng hở không đo được an toàn.
**PercMLP+ không tốt hơn IDM.** Bằng chứng chỉ cho thấy nó "không kém hơn trong biên cho phép": IDM 0.03 % so với 0.07..0.21 % trên log, lưới 0.0 % so với 1.0 %, thế giới phản ứng 10.2 % so với 11.2 %. Nó học nhãn từ IDM nên chỉ có thể tiến gần thầy, không vượt thầy trên chính các thước đo này. Tôi **không** đề xuất thay IDM ngay. Trước hết cần chạy shadow mode (song song với IDM, không điều khiển) trên dữ liệu thật mới, đánh giá thêm họ kịch bản (xe dẫn phanh gấp > 3.6 m/s², tốc độ cao, có nhiễu cảm biến), và chỉ ra lợi ích cụ thể (êm hơn, giống người hơn) đủ bù cho rủi ro.

## Câu 8 · Bài lab này không chứng minh được gì
(1) **Thế giới mô phỏng là giả định của ta, và vòng lặp tự tham chiếu.** IDM vừa là baseline, vừa là "thầy" gắn nhãn cho PercMLP+, vừa là mô hình xe sau phản ứng (T = 1.4 s). PercMLP+ qua cổng một phần vì nó được chấm trong thế giới do chính thầy của nó định nghĩa. Người lái thật có thể phản ứng khác (chậm hơn 0.6 s, phanh khác), và kết luận R6 (11.2 % so với 10.2 %) có thể đổi.
(2) **Một chiều dọc, không cảm biến.** Không có đổi làn, rẽ, người đi bộ. Nhận thức là hoàn hảo, trừ một phép thử nhiễu đơn giản (khoảng cách ±1 m, tốc độ ±0.5 m/s, PercMLP+ 0.07..0.21 %). Lỗi thật như mất dấu, phát hiện sai, trễ cảm biến đều không được kiểm.
(3) **Mẫu nhỏ so với sự kiện hiếm.** Chỉ có 2883 quỹ đạo test và 171 ca phanh. IDM có 1 va chạm, cận trên CI vẫn là 0.2 %, quá thô so với mức an toàn cần cho một xe thật (sự cố trên hàng triệu km). Biên 1–5 điểm của cổng chỉ phù hợp cho bài học.
(4) **Phân bố dữ liệu hẹp.** Argoverse là đô thị, người lái phanh tối đa 3.62 m/s², và các ca ở 20 m/s chỉ có trong mô phỏng. Xe dẫn trong log không phản ứng lại xe thử, và cơ cấu chấp hành lý tưởng (9 m/s², không trễ).
Việc tiếp theo: thu thêm log ở tốc độ cao và phanh gấp; mô phỏng có cảm biến và tác nhân được hiệu chỉnh độc lập với IDM; mở rộng họ kịch bản sang chiều ngang; chạy hardware-in-the-loop; rồi shadow mode trên đường thật với tiêu chí dừng thống kê rõ ràng trước khi phát hành.
