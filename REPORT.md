# REPORT — Lab Day 12

> Điền **bằng lời và số liệu của chính bạn** (lấy từ màn hình và từ `evidence/*.json`). Mỗi câu 4–8 dòng là đủ. Số không có trong bằng chứng của bạn sẽ bị trừ điểm. Báo cáo công khai trên GitHub; giảng viên đối chiếu với `evidence/` và với bài của các bạn khác.

| | |
|---|---|
| Họ và tên | Bùi Văn Quang |
| MSSV (đúng bằng `--student-id` bạn đã chạy) | 2A202602688 |
| GitHub | Quang20040 |
| Seed chia dữ liệu / seed mô hình (xem `evidence/closedloop.json`, mục `student`) | 3914 / 779 |
| Số seed huấn luyện bạn dùng (`--seeds`) | 3 |

## Kết quả cuối

Điền từ `evidence/closedloop.json`, `openloop.json`, `gate.json` (cho mỗi mô hình học, lấy **dải qua các seed**).

| Bộ điều khiển | L2 vòng hở (m) | Va chạm vòng kín trên log (%) | Quyết định cổng |
|---|---|---|---|
| constant-velocity | 1.264 | 19.0% (CI95 [17.7%, 20.5%]) | BLOCK (R2, R3, R4, R6) |
| IDM | 1.531 | 0.07% (CI95 [0.02%, 0.25%]) | (baseline) |
| EgoMLP | 0.772 (0.761 .. 0.788) | 1.7% .. 27.4% (tệ nhất CI95 [25.8%, 29.0%]) | BLOCK (R2, R3, R4, R5, R6) |
| PercMLP | 0.620 (0.619 .. 0.621) | 0.6% .. 7.3% (tệ nhất CI95 [6.4%, 8.3%]) | BLOCK (R2, R3, R4, R5, R6) |
| PercMLP+ (sau khi sửa) | 0.817 (0.803 .. 0.831) | 0.2% .. 0.3% (tệ nhất CI95 [0.14%, 0.55%]) | RELEASE (Đạt R1–R7) |

## Câu 1 · Dữ liệu
Bước 0 bạn phát hiện những vấn đề gì ở dữ liệu? Vì sao không dùng cột `Acc_LV`, và bạn thay bằng gì? Bao nhiêu quỹ đạo bị loại vì trùng, và vì sao chia theo quỹ đạo?

Trong bước 0, quá trình kiểm toán phát hiện cột `Acc_LV` (gia tốc xe dẫn) có hệ số tương quan với đạo hàm vận tốc d(Speed_LV)/dt chỉ đạt 0.0454 (gần như bằng 0), trái ngược với xe theo sau có độ tương quan lên tới 0.9291. Do đó cột `Acc_LV` bị nhiễu nặng và không thể sử dụng trực tiếp; thay vào đó gia tốc xe dẫn được tính bằng đạo hàm vận tốc sau khi đã làm mượt (`a_lead` qua hàm `derive_lead`). Tập dữ liệu gốc gồm 9993 quỹ đạo (992679 hàng) có tới 383 quỹ đạo trùng lặp hoàn toàn (`exact_duplicates_removed = 383`) ở các ID cách xa nhau, đã được loại bỏ để còn 9610 quỹ đạo sạch. Ta bắt buộc phải chia train/test theo quỹ đạo (trajectory-level split) thay vì theo khung hình (frame-level) vì các khung hình kế tiếp trong cùng một quỹ đạo có tính tự tương quan thời gian cực cao; chia theo khung hình sẽ dẫn tới rò rỉ dữ liệu (data leakage) nghiêm trọng khi mô hình học thuộc lòng trạng thái lân cận.

## Câu 2 · Phát lại
Bản refactor float32 khác bit ở bao nhiêu đoạn, vượt dung sai ở bao nhiêu? Thay đổi AEB làm lệch bao nhiêu đoạn, và vì sao lệch ở đúng những khung đó (mở một đoạn ra xem)? Phát theo thời gian thật làm lộ ra điều gì mà phát nhanh nhất che đi?

Bản refactor float32 khác bit ở toàn bộ 600/600 đoạn (`not_bit_identical = 600`), nhưng có 0 đoạn vượt dung sai (`differ_beyond_tol = 0` với sai số tuyệt đối cực đại `max_abs_diff = 4.03e-4` m/s² nhỏ hơn ngưỡng atol=1e-3). Điều này là do tính chất không kết hợp của phép cộng số thực dấu phẩy động (minh họa: (0.1+0.2)+0.3 = 0.6000000000000001 khác 0.1+(0.2+0.3) = 0.6). Thay đổi luật phanh khẩn cấp AEB (`aeb_ttc = 1.5`) làm lệch đúng 5/600 đoạn (các đoạn [61, 226, 438, 476, 506], `max_abs_diff = 1.734` m/s²); các đoạn này bị lệch tại đúng những khung hình mà xe dẫn phanh gấp khiến khoảng cách co hẹp và closing speed tăng cao làm Time-to-Collision rơi xuống dưới 1.5 s, kích hoạt phanh AEB khẩn cấp -6.0 m/s². Phát lại thời gian thực (realtime replay) làm lộ ra 580 khung hình bị rơi (dropped) và 583 khung hình bị trễ (late) trên tổng số 59821 frames, làm đổi lệnh ở 365/600 đoạn (`max_abs_diff = 103.61` m/s²), trong khi chế độ phát nhanh nhất (fastest replay) bỏ qua toàn bộ hạn chót (deadlines) nên hoàn toàn che giấu hiện tượng drop/late này (0 dropped, 0 late).

## Câu 3 · Kịch bản
Lưới của bạn (tốc độ cruise, giới hạn khoảng cách, số ô): bao nhiêu % ô **không ai tránh được**? Độ trễ phản ứng 0 → 0,6 s làm vùng đâm của IDM tăng bao nhiêu? Log có phủ vùng khó không? (dẫn giới hạn phanh mạnh nhất của người lái trong log, và phần lưới đòi hỏi hơn thế)

Lưới kịch bản cut-in được thiết lập ở vận tốc cruise `v_ego = 10.0` m/s, khoảng cách gap từ 2.76 m đến 59.31 m, tốc độ tiến gần dv từ 0.0 đến 10.0 m/s gồm 2700 ô (60x45). Ở độ trễ phản ứng `t_react = 0.0` s, có 0.96% số ô (26 ô) là va chạm bất khả kháng theo vật lý (`unavoidable`). Khi độ trễ phản ứng tăng lên `t_react = 0.6` s, vùng bất khả kháng tăng vọt lên 5.15% số ô (139 ô), tức tăng thêm 4.19 điểm phần trăm (gấp ~5.3 lần); IDM đâm đúng bằng diện tích bất khả kháng này với `mismatch_cells_away_from_frontier = 0`. Dữ liệu log không hề bao phủ được các vùng khó: con người trong log phanh mạnh nhất ở phân vị 99.9% chỉ đạt 3.62 m/s² (`human_brake_limit_p99.9 = 3.62` m/s²), và có tới 5.22% số ô trên lưới đòi hỏi lực phanh vượt quá giới hạn của con người. Hơn nữa, toàn bộ 614 sự kiện phanh gấp khai phá từ log (chỉ chiếm 6.39% số quỹ đạo) đều nằm ở vùng an toàn (gap lớn, dv nhỏ) chứ không bao phủ các góc tới hạn (corner cases) của lưới cut-in.

## Câu 4 · Vòng hở và vòng kín
Dán bảng L2 và tỷ lệ va chạm của bốn bộ điều khiển (kèm hạng ở mỗi cách chấm). Cặp nào đổi thứ tự? Vì sao IDM có L2 như vậy mà vẫn lái tốt? Vì sao EgoMLP có L2 tốt hơn constant-velocity mà vẫn đâm?

Bảng đối chiếu thứ hạng giữa vòng hở và vòng kín:
1. PercMLP: L2 vòng hở = 0.620 m (Hạng 1) | Va chạm vòng kín = 2.84% (worst seed 7.28%) (Hạng 2).
2. EgoMLP: L2 vòng hở = 0.772 m (Hạng 2) | Va chạm vòng kín = 12.14% (worst seed 27.37%) (Hạng 3).
3. const-vel: L2 vòng hở = 1.264 m (Hạng 3) | Va chạm vòng kín = 19.04% (Hạng 4).
4. IDM: L2 vòng hở = 1.531 m (Hạng 4) | Va chạm vòng kín = 0.07% (chỉ 2/2883 xe va chạm) (Hạng 1).
Hiện tượng đảo ngược thứ hạng (ranking inversion) xảy ra ở 3 cặp: `('const-vel', 'IDM')`, `('IDM', 'EgoMLP')`, và `('IDM', 'PercMLP')`. IDM có L2 kém nhất (1.531 m) vì nó không tối ưu hóa để khớp từng milimet hành vi của một tài xế cụ thể trong log, nhưng lại lái an toàn nhất (0.07% va chạm) nhờ cấu trúc điều khiển phản hồi vòng kín (closed-loop feedback) duy trì khoảng cách mong muốn $s^*$. Ngược lại, EgoMLP đạt L2 tốt hơn const-vel (0.772 m vs 1.264 m) vì dự đoán tốt vận tốc khi xe chạy êm, nhưng hoàn toàn thiếu cảm biến nhận thức phía trước (`gap` và $v_{lead}$); khi vào vòng kín gặp xe trước phanh, sai số tích lũy (compounding error) khiến nó đâm tới 12.14% (tệ nhất 27.37%).

## Câu 5 · Một lần chạy chưa đủ
EgoMLP dao động bao nhiêu giữa các seed? Khoảng tin cậy Wilson nói gì? Nếu bạn chỉ chạy đúng một seed thì có thể báo kết quả nào, và vì sao điều đó nguy hiểm?

Tỷ lệ va chạm vòng kín của EgoMLP biến thiên cực kỳ dữ dội qua 3 seed: từ 1.73% ở seed 779 (50/2883 va chạm), lên 7.32% ở seed 781 (211/2883 va chạm), và đạt đỉnh 27.37% ở seed 780 (789/2883 va chạm), chênh lệch giữa seed tốt nhất và tệ nhất lên tới 25.64 điểm phần trăm. Khoảng tin cậy Wilson 95% của seed tốt nhất là [1.32%, 2.28%] hoàn toàn tách rời (disjoint), không hề giao thoa với khoảng tin cậy của seed tệ nhất là [25.77%, 29.02%]. Điều này chứng tỏ sự chênh lệch bắt nguồn từ việc mô hình hội tụ vào các cực tiểu cục bộ (local minima) rất khác nhau về độ ổn định vòng kín chứ không phải do nhiễu mẫu ngẫu nhiên. Nếu chỉ chạy duy nhất seed 779, kỹ sư có thể báo cáo sai lầm rằng EgoMLP rất an toàn với tỷ lệ va chạm chỉ 1.7%, nhưng thực tế hệ thống có thể đối mặt với rủi ro đâm xe lên tới hơn 27% khi triển khai thực tế. Do đó, kiểm thử an toàn bắt buộc phải đo lường trên nhiều seed và căn cứ vào cận trên của khoảng tin cậy Wilson ở seed xấu nhất.

## Câu 6 · Tác nhân phản ứng
Dán bảng "xe sau phát lại" và "xe sau phản ứng". `T` bạn hiệu chỉnh là bao nhiêu, xe sau chân thực đến mức nào so với xe không phản ứng? Vì sao IDM bị tính "va chạm giả" khi xe sau chỉ phát lại, và điều này làm đổi kết luận về bộ điều khiển ra sao?

Thời khoảng cách hiệu chỉnh tốt nhất của IDM là `T = 1.4` s với khoảng cách Wasserstein-1 $W_1 = 0.3558$ so với phân bố headway thực tế của con người (trung vị 1.87 s), chân thực hơn đáng kể so với tác nhân không bao giờ phản ứng ($W_1 = 0.8649$).
Bảng va chạm trên lưới cut-in (t_react = 0.6 s) giữa xe sau phát lại (replay) và xe sau phản ứng (reactive):
- const-vel: Replay = 73.37% (Rear: 0.0%) | Reactive = 73.37% (Rear: 0.0%).
- IDM: Replay = 70.07% (Rear: 64.93%) | Reactive = 5.15% (Rear: 0.0%).
- EgoMLP: Replay = 100.0% (Rear: 40.04%) | Reactive = 60.11% (Rear: 0.0%).
- PercMLP: Replay = 76.41% (Rear: 46.33%) | Reactive = 31.26% (Rear: 0.0%).
- PercMLP+: Replay = 69.63% (Rear: 64.44%) | Reactive = 5.19% (Rear: 0.0%).
Khi xe sau chỉ phát lại log (replay), nó không biết xe thử (ego) phanh gấp để tránh xe cắt vào nên giữ nguyên tốc độ và đâm vào đuôi xe thử trong 64.93% số ô đối với IDM. Đây là "va chạm giả" (simulator artifact) do môi trường mô phỏng thụ động gây ra. Hiện tượng này làm sai lệch nghiêm trọng kết luận: IDM từ chỗ trông nguy hiểm ngang const-vel (70.07% vs 73.37%) thực chất lại loại bỏ sạch 100% va chạm đuôi khi xe sau có phản ứng, đưa tỷ lệ va chạm về đúng 5.15% (bằng đúng biên vật lý không thể tránh).

## Câu 7 · Cổng phát hành và sửa
Quy tắc nào chặn PercMLP và nguyên nhân gốc là gì (dẫn số)? Bạn sửa bằng gì và vì sao nó chữa đúng nguyên nhân? Sau khi qua cổng, PercMLP+ có **tốt hơn** IDM không? Nó có nên được phát hành không, và bạn đề xuất làm gì trước khi quyết định?

PercMLP ban đầu bị cổng phát hành CHẶN (BLOCK) bởi 5 quy tắc: R2 (va chạm 7.3%, cận trên 8.3% > baseline 0.1% + 1.0%), R3 (lát cắt phanh 17.9%, cận trên 24.3% > baseline 0.6% + 5.0%), R4 (lưới kịch bản 26.1% > 2.0%), R5 (dao động seed 0.6%..7.3% > 1.0%), và R6 (thế giới phản ứng 31.3% > baseline 5.1% + 2.0%). Hai nguyên nhân gốc là hiện tượng trôi dạt phân phối (covariate shift) tích lũy sai số khi tự lái và sự vắng bóng hoàn toàn các kịch bản cut-in ngặt nghèo trong log. Ta sửa bằng cách bổ sung 20000 kịch bản cut-in mô phỏng (197080 cửa sổ) và áp dụng thuật toán DAgger qua 2 vòng rollout thu thập trạng thái chính mô hình tự đi vào kèm nhãn dự báo từ IDM. Sau khi sửa, PercMLP+ đạt RELEASE với cả 7 quy tắc vượt qua (va chạm vòng kín giảm còn 0.2%..0.3%, cận trên 0.5% đạt R2; diện tích lưới cut-in giảm về 0.0% đạt R4). Tuy nhiên, PercMLP+ không tốt hơn IDM (va chạm 0.2%..0.3% so với 0.07% của IDM; ở tốc độ 20 m/s chưa từng thấy khi train, PercMLP+ đâm 0.4%..1.3% trong khi IDM là 0.0%). PercMLP+ chưa nên phát hành thay thế IDM; cần thử nghiệm mở rộng đa làn 2D và kiểm thử trên xe thật với tài xế an toàn trước khi đưa ra quyết định.

## Câu 8 · Bài lab này không chứng minh được gì
Chọn **ít nhất ba** giới hạn quan trọng nhất đối với việc quyết định phát hành và giải thích vì sao mỗi cái làm kết luận của bạn kém chắc chắn. Điều gì cần làm tiếp để thu hẹp khoảng cách đến thế giới thật?

Ba giới hạn quan trọng nhất của bài lab đối với quyết định phát hành:
1. Mô hình hóa thuần túy 1 chiều dọc (1-D Longitudinal only): Hệ thống hoàn toàn không có khả năng đánh lái (lateral control) hoặc chuyển làn né tránh. Trong thực tế, nhiều va chạm dọc khẩn cấp được tài xế hóa giải an toàn bằng cách chuyển làn hoặc lách chướng ngại vật; việc ép mô hình chỉ được phanh khiến đánh giá rủi ro bị sai lệch so với vận hành xe thực tế.
2. Cảm biến nhận thức lý tưởng (Ideal Perception & Lack of Latency): Bài lab cung cấp hoàn hảo khoảng cách `gap` và vận tốc xe trước $v_{lead}$ không có độ trễ hệ thống và chỉ có nhiễu Gaussian nhân tạo tối thiểu (gap ±1.0 m, speed ±0.5 m/s). Trong thực tế xe tự hành, camera/radar/LiDAR chịu ảnh hưởng nặng nề bởi thời tiết xấu, vật cản che khuất (occlusion), bounding box chập chờn và độ trễ xử lý của mô hình tri giác (>100 ms) dễ gây cộng hưởng mất ổn định pha điều khiển.
3. Tác nhân xung quanh được đơn giản hóa cao độ (Simplified Agent Behavior): Xe dẫn phát lại từ log không hề phản ứng với xe thử, trong khi xe sau chỉ là một luật IDM đơn lẻ. Thế giới thực là một hệ thống trò chơi tương tác đa tác nhân phức tạp (interactive game-theoretic environment), nơi con người có phản xạ giật mình, phanh nhấp nhả, hoặc chủ động nhường đường.
Để thu hẹp khoảng cách đến thế giới thực, bước tiếp theo cần triển khai mô phỏng 2D/3D tích hợp cả điều khiển dọc và ngang, đưa vào pipeline cảm biến mô phỏng chân thực (CARLA/nuPlan), và tiến hành thử nghiệm closed-loop trên đường thử kín (proving ground) với tài xế an toàn ngồi trong xe giám sát.
