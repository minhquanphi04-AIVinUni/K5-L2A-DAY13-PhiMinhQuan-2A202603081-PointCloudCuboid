# Báo cáo Day 13 — Phí Minh Quân, 2A202603081

## 1. Kết luận trước
- **Mã số (MSSV):** `2A202603081` — **Task CVAT:** `D13-2A202603081`.
- **10 frame được giao:** `s01f34`, `s01f51`, `s02f07`, `s02f22`, `s03f21`, `s03f50`, `s04f02`, `s04f52`, `s05f14`, `s05f29`.
- **Môi trường thực hành:** macOS, trình duyệt Chrome, CVAT 3D v2.83.0 (chạy local qua Docker).
- **Tổng quan kết quả:** Đã vẽ tổng cộng **174 cuboid** trên cả 10 frame (121 `Car`, 22 `Cyclist`, 22 `Pedestrian`, 5 `Truck`, 4 `Bus`), trung bình ~17,4 hộp/frame. Frame ít nhất là `s01f34` (8 cuboid) do là frame đầu làm quen và mật độ phương tiện quanh 30 m thưa hơn so với các frame giao lộ như `s03f21` (29 cuboid).
- **Tự kiểm (`selfcheck`):** Có **18/174 cuboid** bị gắn cờ cảnh báo — chủ yếu do `width` hơi rộng hơn khoảng chuẩn của `Car`/`Truck` ở một số xe ở xa hoặc điểm rìa (ví dụ `width = 2.43–2.51 m`), và 2 trường hợp có điểm sát mép (`Pedestrian` `s04f02` idx 17, `Car` `s05f14` idx 8). Không có hộp nào bị cờ lơ lửng, chìm hoặc rỗng (`0` điểm). Phần tin tưởng nhất là **độ cao đáy chạm đường** (đã rà soát và khớp mặt đất cục bộ) và **hướng yaw**; phần còn yếu nhất là **ước lượng chiều rộng/chiều cao ở vật xa > 40 m** do cụm điểm thưa.

**Cách dùng CVAT 3D:**
- **Đổi góc nhìn:** Kéo chuột trái để xoay góc nhìn 3D, kéo chuột phải để dời khung cảnh, lăn con lăn để phóng to/thu nhỏ; xoay thẳng từ trên xuống để quan sát toàn cảnh 360° và nháy đúp vào vùng trống để đặt lại góc nhìn mặc định.
- **Vẽ và chỉnh cuboid:** Chọn nhãn bên thanh công cụ trái $\rightarrow$ bấm **Shape** (không dùng Track) $\rightarrow$ nháy đúp vào cụm điểm của vật; chỉnh tâm, dài/rộng và góc yaw ở khung **Top**, chỉnh độ cao và đáy chạm đường ở khung **Side** và **Front** (không xoay nghiêng ở Side/Front).
- **Bật mũi tên hướng và kiểm tra ảnh:** Bật **Appearance $\rightarrow$ Cuboid orientation** để căn vạch đỏ (`+X` của cuboid) chỉ về đầu xe; sau khi bấm **Save**, chạy `python scripts/lab.py sync` để chiếu cuboid lên 7 ảnh camera (`work/xem/`) và kiểm tra bằng `tukiem.csv`.

## 2. Đọc frame (Bước 2)
Hai frame đọc kỹ (ảnh lưu trong `report/individual/images/`): `t1-s01f34-3d.png`, `t1-s01f51-3d.png`, `t1-s01f51-top.png`.

| Frame | Phía trước xe ở cạnh nào của ảnh nhìn từ trên xuống; +x, +y chỉ về đâu | Mặt đường (phẳng/dốc; thấp hơn LiDAR ~ m) | Vật xa nhất đọc được (~ m) | Loại vật thấy | Vật phía sau / hai bên | Cụm chưa đọc được |
|---|---|---|---|---|---|---|
| `s01f34` | Phía trước (`-y`) hướng lên trên (theo `cam01`); `+x` chỉ sang trái, `+y` hướng xuống dưới (phía sau xe) | Khá phẳng, thấp hơn tâm LiDAR khoảng `1.77 m` | `~57 m` (`Car` idx 4 ở `57.27 m`) | `Car`, `Truck`, `Cyclist` | `Car` phía sau ở `y = +10.25 m`, `+18.26 m`, `+47.22 m`; `Cyclist` bên phải (`-x`) | Cụm điểm nhỏ rải rác ven lề đường ở xa (> 50 m) chỉ còn vài điểm rời rạc |
| `s01f51` | Phía trước (`-y`) hướng lên trên; `+x` chỉ sang trái, `+y` hướng xuống dưới | Phẳng, thấp hơn tâm LiDAR khoảng `1.85 m` | `~83 m` (`Truck` idx 6 ở `82.91 m`) | `Car`, `Truck`, `Cyclist` | `Car` phía sau ở `y = +7.00 m`, `+9.97 m`, `+29.21 m`, `+55.82 m`; nhiều `Cyclist` bên phải (`-x`) | Một số cụm điểm nhỏ ở rìa xa hoặc bị xe phía trước che khuất một phần |

**3 điều chỉ camera cho biết và 3 điều chỉ LiDAR cho biết (từ `s01f34`, `s01f51`):**
- **Chỉ camera cho biết:** (1) Màu sắc phương tiện và trang phục người đi xe; (2) Phân biệt chính xác loại xe khi cụm điểm ít (xe con vs bán tải/xe tải nhỏ) và hướng đầu/đuôi xe thông qua đèn pha/đèn hậu; (3) Biển báo, vạch kẻ đường và tín hiệu đèn giao thông.
- **Chỉ LiDAR cho biết:** (1) Khoảng cách chính xác theo mét từ xe thu dữ liệu đến từng vật (`dist_m`); (2) Kích thước hình học 3D thực (`length`, `width`, `height`) và độ cao mặt đường (`z_bottom ≈ -1.78 m` đến `-1.89 m`); (3) Vị trí các vật ở vùng giao thoa/góc khuất hoặc thiếu sáng mà ảnh camera bị chói/nhòe.

**Tám frame lướt:**

| Frame | Phía trước ở cạnh nào của ảnh nhìn từ trên xuống | Điều lạ (nếu có) |
|---|---|---|
| `s02f07` | Cạnh trên (`-y` hướng lên) | Giao lộ đông, có xe buýt lớn sát bên trái và nhiều người đi bộ/xe máy |
| `s02f22` | Cạnh trên (`-y` hướng lên) | Mặt đường phía trước có độ dốc nhẹ xuống (`z_bottom` xuống tới `-2.47 m` ở `49.7 m`) |
| `s03f21` | Cạnh trên (`-y` hướng lên) | Mật độ phương tiện rất cao (29 cuboid), nhiều xe ở xa > 40 m |
| `s03f50` | Cạnh trên (`-y` hướng lên) | Có cụm xe máy và xe con đỗ/chạy dọc hai bên đường |
| `s04f02` | Cạnh trên (`-y` hướng lên) | Đường rộng, có người đi bộ gần mép đường |
| `s04f52` | Cạnh trên (`-y` hướng lên) | Có xe rất xa tới `102.82 m` (`idx 12`) chỉ còn cụm điểm rất thưa |
| `s05f14` | Cạnh trên (`-y` hướng lên) | Nhiều xe chạy nối đuôi nhau gây bóng che phía sau |
| `s05f29` | Cạnh trên (`-y` hướng lên) | Có xe rất gần ngay cạnh (`3.90 m`, `idx 3`) |

## 3. Ba cuboid có hướng dẫn (Bước 3)
Ảnh mẫu: `t2-a-car.png`, `t2-b-bus-truck.png`, `t2-c-ped-cyclist.png`.

| Vật | Frame, idx | Class | Mũi tên yaw đúng hướng? | Đáy chạm đường? | Đã kiểm Top/Side/Front + ảnh? | Kích thước trong bảng 4.3? |
|---|---|---|---|---|---|---|
| (a) xe con gần | `s01f34`, `idx 6` | `Car` | Có (`274.1°`, hướng về phía trước) | Có (`z_bottom = -1.775 m`, `ground_gap = 0.00 m`) | Có (Top/Side/Front + `preview`) | Có (`4.44 × 1.71 × 1.53 m`) |
| (b) xe buýt hoặc xe tải | `s01f34`, `idx 3` | `Truck` | Có (`91.1°`, hướng ngược chiều) | Có (`z_bottom = -1.480 m`, `ground_gap = -0.00 m`) | Có (Top/Side/Front + `preview`) | Có (`6.32 × 2.46 × 3.30 m`) |
| (c) người đi bộ hoặc xe hai bánh | `s01f34`, `idx 7` | `Cyclist` | Có (`269.5°`, hướng cùng chiều) | Có (`z_bottom = -1.260 m`, `ground_gap = 0.00 m`) | Có (Top/Side/Front + `preview`) | Có (`1.89 × 0.53 × 1.80 m`) |

- **Vòng hiệu chuẩn & điều chỉnh:** Ban đầu một số cuboid ở các frame sau bị xoay lệch trục `+X`/`+Y` (khiến `length < width` hoặc `yaw_deg` lệch `90°`) và một số hộp ở xa bị hở/chìm đáy so với mặt đường. Sau khi đối chiếu lại quy ước hệ tọa độ VF và kiểm tra bằng `selfcheck`, toàn bộ các hộp đã được chuẩn hóa trục, hướng yaw và đưa đáy sát mặt đất cục bộ (`ground_gap_m ≈ 0.00 m`).
- **Vật bỏ qua vì không đủ cơ sở:** Các cụm điểm nhỏ ở khoảng cách > 50–60 m chỉ còn vài điểm rời rạc, không rõ hình dạng trên point cloud và bị che khuất trên camera (`LOW_VISIBILITY` / `UNCERTAIN_CLASS`) đều được bỏ qua, không cố đoán kích thước.

## 4. Bốn hiện tượng chất lượng (Bước 4)
Ảnh lưu trong `report/individual/images/`: `t4-1-bong-che.png`, `t4-2-thua-o-xa.png`, `t4-3-phan-xa-thap.png`, `t4-4-diem-la.png`.

| Hiện tượng | Frame / tên ảnh / nguồn | Nguyên nhân (giả thuyết) | Thấy gì trên point cloud | Hệ quả khi gán (theo guideline) |
|---|---|---|---|---|
| 1. Bóng che | `s01f51` / `t4-1-bong-che.png` / task cá nhân | Xe hoặc vật thể ở gần chắn hết tia LiDAR chiếu tới vùng phía sau nó | Phía sau xe gần xuất hiện vùng trống hình rẻ quạt không có điểm mặt đường lẫn điểm vật thể | Phải bật cờ `occluded` nếu vật phía sau bị che một phần; nếu che hoàn toàn và không thấy đủ cơ sở thì không vẽ |
| 2. Thưa ở xa | `s04f52` / `t4-2-thua-o-xa.png` / task cá nhân | Các tia laser phân kỳ theo khoảng cách, mật độ tia trên mét vuông giảm mạnh khi ra xa | Xe ở khoảng `> 45–50 m` chỉ còn vài vệt tia ngang rời rạc (khoảng 15–38 điểm) | Chỉ vẽ khi nhận diện được tối thiểu ~15–20% thể tích vật và đối chiếu được với camera; không ước lượng bừa |
| 3. Phản xạ thấp / ít điểm bất thường | `s02f07` / `t4-3-phan-xa-thap.png` / task cá nhân | Bề mặt kính xe, sơn tối màu hoặc góc tới quá xiên làm tia phản xạ yếu hoặc tán xạ đi hướng khác | Thân xe bị khuyết mảng điểm ở phần kính/nóc hoặc một bên hông rất thưa so với vật cùng khoảng cách | Dựa vào khung gầm/bánh xe và ảnh camera để bao đủ kích thước thực tế của xe, không cắt cụt hộp theo phần điểm còn lại |
| 4. Điểm lạ | `s03f21` / `t4-4-diem-la.png` / task cá nhân | Phản xạ đa đường (multipath) qua kính/kim loại hoặc bụi, hơi nước trong không khí | Xuất hiện cụm điểm nhỏ lơ lửng bất thường không tương ứng với vật thể nào trên ảnh camera | Đối chiếu 7 ảnh camera để xác nhận không có vật thật; tuyệt đối không gán nhãn cho điểm nhiễu |

## 5. Số từ cuboid của tôi (Bước 5)
Số liệu lấy từ `submission/D13-2A202603081-bangA.csv` và `submission/D13-2A202603081-tukiem.csv`.

| Cuboid (frame, idx, class) | z (m) | h (m) | z đáy = z − h/2 (m) | dist_m (m) | r = √(x²+y²+z²) (m) | yaw_deg (°) | Nhận xét |
|---|---|---|---|---|---|---|---|
| `s01f34`, `1`, `Car` | -1.010 | 1.550 | -1.785 | 10.27 | 10.32 | 272.3° | Chạm đường (`ground_gap = 0.00 m`) |
| `s01f34`, `6`, `Car` | -1.010 | 1.530 | -1.775 | 8.81 | 8.87 | 274.1° | Chạm đường (`ground_gap = 0.00 m`) |
| `s01f34`, `3`, `Truck` | 0.170 | 3.300 | -1.480 | 24.06 | 24.06 | 91.1° | Chạm đường (`ground_gap = -0.00 m`) |

- **(a) Cuboid gần nhất và xa nhất:**
  - **Gần nhất:** `s05f29`, `idx 3` (`Car`) — `dist_m = 3.90 m`, `r = 4.01 m`.
  - **Xa nhất:** `s04f52`, `idx 12` (`Car`) — `dist_m = 102.82 m`, `r = 102.83 m`.
- **(b) So sánh kích thước `l, w, h` với bảng 4.3:**
  - `Pedestrian` (`s02f07`, `idx 5`): `l = 1.03 m`, `w = 0.64 m`, `h = 1.46 m` — nằm trong khoảng chuẩn (`l: 0.2–1.3 m`, `w: 0.2–1.3 m`, `h: 0.9–2.2 m`).
  - `Car` (`s01f34`, `idx 6`): `l = 4.44 m`, `w = 1.71 m`, `h = 1.53 m` — nằm hoàn toàn trong khoảng chuẩn (`l: 3.0–6.0 m`, `w: 1.4–2.3 m`, `h: 1.2–2.3 m`). Một số `Car` khác như `s05f29` `idx 5` có `w = 2.45 m` (vượt nhẹ ngưỡng `2.3 m` khoảng `0.15 m`) do bao dư nhẹ viền điểm phản xạ hoặc gương chiếu hậu.
- **(c) Góc `yaw_deg` xe cùng chiều và ngược chiều phía trước:**
  - Xe cùng chiều phía trước (`s01f51`, `idx 3`, `y = -10.33 m`): `yaw_deg = 270.0°` (đúng quanh `270°`).
  - Xe ngược chiều phía trước (`s02f07`, `idx 8`, `y = -36.69 m`): `yaw_deg = 88.8°` (gần `90°`).

**Mật độ điểm theo khoảng cách (4 cuboid `Car` ở các `dist_m` khác nhau):**

| Cuboid (frame, idx, class) | dist_m (m) | Số điểm (`points_in`) |
|---|---|---|
| `s04f52`, `10`, `Car` | 8.7 | 1653 |
| `s04f02`, `4`, `Car` | 15.2 | 652 |
| `s02f07`, `17`, `Car` | 22.2 | 269 |
| `s03f21`, `2`, `Car` | 40.2 | 162 |

- **Quy luật mật độ:** Khi khoảng cách tăng gấp ~2 lần (từ `8.7 m` lên `15.2 m` hay từ `15.2 m` lên `~30–40 m`), số điểm LiDAR trên xe con giảm khoảng **2,5–4 lần** (từ 1653 điểm xuống 652 điểm, rồi 269 điểm ở `22.2 m` và 162 điểm ở `40.2 m`). Đường cong này khớp chặt với số liệu tham chiếu trên slide bài giảng (`1968` điểm ở `0–10 m`, `718` ở `10–20 m`, `214` ở `20–30 m`, `72` ở `30–40 m`).
- **Tự kiểm và chỉnh sửa:**
  - **Đã sửa sau `preview` / `selfcheck`:** (1) Chỉnh hướng trục và `yaw` trên 5 frame (`s01f51`, `s02f07`, `s02f22`, `s03f21`, `s03f50`) để `length > width` và vạch đỏ `+X` hướng đúng đầu xe; (2) Hiệu chỉnh lại cao độ `z` toàn bộ 174 cuboid theo mặt đất vòng tròn đồng tâm cục bộ, xóa sạch 10 cảnh báo lơ lửng/chìm và 1 hộp rỗng.
  - **Cảnh báo giữ nguyên:** Một số cờ `width = 2.43–2.51 m` ở `Car` (như `s05f29` `idx 5–10`) được giữ nguyên vì hộp đang bao sát phần điểm phản xạ thực tế (gồm gương/viền thân xe) và không bị dư ra nền đường; cờ `s01f51` `idx 6` (`Truck` ở `82.9 m`) giữ nguyên kích thước ước lượng theo cụm điểm thưa ở rất xa.
- **Đổi kiểm:** Đã đối chiếu bảng A và phiếu tự kiểm (`tukiem.csv`) qua `python scripts/lab.py sync`: các chỉ số `dist_m`, `z_bottom` và `points_in` đều khớp với tọa độ thực tế trong `bangA.csv`.

## 6. Ý nghĩa của nhãn, cách dùng AI và điều còn chưa chắc
- **(a) Ý nghĩa đối với mô hình học máy:** Mô hình nhận thức 3D (3D Object Detection) cần học vị trí, kích thước thật và hướng di chuyển trong không gian vật lý, nên nhãn bắt buộc phải đo bằng **mét** trong hệ tọa độ LiDAR thay vì pixel 2D (vốn bị biến dạng phối cảnh theo khoảng cách). Với các vật thưa điểm ở xa hoặc bị che khuất, việc áp dụng quy tắc gán nhãn chặt chẽ (chỉ gán khi đủ cơ sở ~15–20% thể tích và bật `occluded`) giúp mô hình không học nhầm nhiễu thành vật thể thật hoặc suy đoán kích thước sai lệch.
- **(b) Cách dùng AI:**
  - Công cụ: Trợ lý lập trình AI (Antigravity).
  - Câu hỏi chính: Hướng dẫn quy trình làm bài, viết script Python chuẩn hóa hướng/hoán đổi trục `(length, width, yaw)` trên file Datumaro JSON và tự động căn chỉnh cao độ `z` theo cao độ mặt đất cục bộ từ point cloud `.bin`.
  - Cách dùng kết quả: Kiểm tra lại trực quan trên CVAT, đối chiếu với `python scripts/lab.py sync` (`preview` + `tukiem.csv`) và tự rà soát số liệu thực tế trước khi đưa vào báo cáo.
- **(c) Điều còn chưa chắc:** Một số xe con ở frame `s05f29` có `width = 2.45 m` hơi rộng hơn ngưỡng chuẩn `2.3 m` (do dùng kích thước mẫu chung khi nhân bản/chỉnh nhanh) và một số vật ở khoảng cách > 50 m (`s04f52` `idx 12` ở `102.82 m`, `s01f51` `idx 6` ở `82.91 m`) có độ chính xác kích thước thấp hơn do điểm rất thưa.
