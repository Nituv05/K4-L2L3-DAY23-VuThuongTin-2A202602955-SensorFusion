# Báo cáo bài nộp — Day 23 Sensor Fusion Lab

> Trạng thái: E–H đã hoàn thành và vượt qua toàn bộ unit test. CP5 chưa chạy được vì bản Waymo trên Drive của lớp đang vượt hạn mức tải. Chưa có kết quả tracking để kết luận chất lượng; bài chưa sẵn sàng nộp.

## Thông tin học viên

- Họ tên: Vũ Thượng Tín
- MSSV: 2A202602955
- Email: Nituv05@users.noreply.github.com (địa chỉ GitHub noreply dùng cho commit)
- Link repo (fork): https://github.com/Nituv05/K4-L2L3-DAY23-VuThuongTin-2A202602955-SensorFusion
- Commit hash nộp (`git rev-parse HEAD`): Chưa chốt; lấy commit cuối sau khi hoàn tất CP5, báo cáo và push.

Repo đã được đổi đúng mẫu `K4-L2L3-DAY23-VuThuongTin-2A202602955-SensorFusion`.

## Tóm tắt kết quả

- Unit test: **128 passed, 0 failed, 0 xfailed**; chạy bằng Python 3.12.9 và protobuf 6.33.6 trong môi trường riêng.
- `fusion_mode` (bắt buộc `compare`), `frames`, `segment`, `seed`: Chưa có kết quả. Cấu hình lần chạy chính thức là `compare`, `[0, 198]`, `training_segment-1005081002024129653_5313_150_5333_150_with_camera_labels.tfrecord`, seed `0`.
- `detection.precision`, `detection.recall`, `detection.tp/fp/fn`: Chưa chạy Waymo, chưa có số liệu.
- `tracking.lidar.rmse`, `matches`, `sum_sq_err`, `ghost_track_frames`, `missed_gt_frames`, `mean_confirmed_tracks`: Chưa chạy Waymo, chưa có số liệu.
- `tracking.fused.rmse`, `matches`, `sum_sq_err`, `ghost_track_frames`, `missed_gt_frames`, `mean_confirmed_tracks`: Chưa chạy Waymo, chưa có số liệu.
- Giải thích khác biệt hai mode, đọc RMSE cùng số ghép và ghost/miss: Chưa kết luận khi chưa có artifacts thực tế. Sau khi chạy, đối chiếu RMSE, matches, ghost/miss, `precision_track = matches / (matches + ghost_track_frames)` và `coverage = matches / det_tp` ở cả hai mode.

Lệnh tái lập từ root repo, sau khi kích hoạt môi trường và có dữ liệu/weights:

```bash
python -m pytest student/tests -q
fusion-run-lab --config student/config/paths.yaml --fusion compare --seed 0
python tools/check_submission.py
```

RMSE được tính bằng `sqrt(sum_sq_err / matches)` trên sai số vị trí **3D** của confirmed tracks ghép một-một với GT xe, gate XY **2 m**. Khi không có cặp ghép, RMSE là `null`, không phải 0. RMSE thấp nhưng ít cặp ghép hoặc nhiều ghost chưa đủ chứng minh tracker tốt.

Camera dùng tâm hộp **ground-truth 2D FRONT** cộng nhiễu theo seed; không chạy camera detector. Kết quả fusion chỉ đánh giá bộ lọc với nguồn đo mô phỏng này.

Lần chạy compare phải sinh đủ `metrics.json`, `grade_run.log`, `metrics_lidar.json`, `metrics_fused.json`, `grade_run_lidar.log`, `grade_run_fused.log` trong `student/artifacts/`. Mỗi `(mode, frame)` có một record; `matches + ghosts == confirmed` và `matches + misses == valid_gt`. Tổng đếm, tổng bình phương sai số và trung bình confirmed trong log phải khớp metrics. Với 199 frame, log gộp có 398 record.

## Giải thích ngắn (Parts E–H)

1. **Khác biệt đo LiDAR 3D và camera 2D trong EKF (`z`, `R`)?**
   LiDAR có `z = [x_s, y_s, z_s]^T` trong hệ cảm biến, đơn vị mét, và `R` kích thước 3×3 chứa phương sai đo theo ba trục. Camera có `z = [u, v]^T`, đơn vị pixel, và `R = diag(sigma_cam_i², sigma_cam_j²)` kích thước 2×2. Cả hai cập nhật cùng trạng thái 6D `[px, py, pz, vx, vy, vz]^T`. Trong [kalman.py](workspace/kalman.py), innovation là `z - h(x)`, `S = H P H^T + R`; LiDAR có phép đo tuyến tính, còn camera chiếu pinhole phi tuyến và dùng Jacobian do platform cung cấp. [camera_fusion.py](workspace/camera_fusion.py) đổi vị trí sang hệ cảm biến trước khi tính `u = c_i - f_i*y_s/x_s`, `v = c_j - f_j*z_s/x_s`.

2. **Vì sao cần gating Mahalanobis trước khi gán?**
   Trong [association.py](workspace/association.py), `d² = gamma^T S^(-1) gamma` đo residual theo độ bất định của track và cảm biến. Cổng `chi2.ppf(gating_threshold, dim_meas)` loại cặp không phù hợp trước khi chọn greedy, tránh buộc ghép các đo ở xa. Khi `P` lớn, residual cùng độ lớn có thể cho Mahalanobis nhỏ hơn; Euclidean không xét độ bất định này. Code kiểm tra FOV trước khi tính khoảng cách để không chiếu camera ở độ sâu không hợp lệ, và dùng giải hệ tuyến tính thay vì tạo nghịch đảo tường minh.

3. **Pipeline là track-then-fuse hay fuse-then-track?**
   Đây là **track-then-fuse**: một danh sách track, predict một lần mỗi frame, association/update LiDAR, quản lý track, rồi association/update camera trên cùng track. Thứ tự thể hiện trong `platform/fusion_lab/scripts/run_lab.py`; [association.py](workspace/association.py) chỉ update và không predict lại. Waymo Frame được coi là một tick đồng bộ; measurement của hai cảm biến trong cùng frame có `t = frame_index * dt`. Log chưa sinh nên chưa có dẫn chứng theo frame. Khi có log, record `mode = lidar/fused` cho phép đối chiếu confirmed, matches, ghosts và misses; log tổng hợp không chứa riêng sự kiện từng sensor update nên cần kết hợp với code để xác nhận thứ tự.

4. **Nếu camera lệch calibration, triệu chứng gì trên innovation/residual?**
   Sai extrinsic làm vị trí trong hệ camera sai; sai intrinsic làm pixel dự đoán sai. Residual có thể lệch có hệ thống theo chiều ảnh, Mahalanobis tăng và nhiều cặp bị gating loại. Nếu sai lệch vẫn nằm trong gate, camera update có thể kéo trạng thái khỏi vị trí đúng và làm RMSE fused tăng. Đây là giải thích từ mô hình; chưa chạy thí nghiệm calibration và không báo cáo số liệu bonus.

5. **Vì sao `associate_and_update(..., sensor)` cần sensor tường minh ở frame rỗng?**
   Khi `meas_list` rỗng, không thể suy ra modality từ measurement đầu tiên. Sensor tường minh giúp vẫn gọi `manager.manage_tracks(unassigned_tracks, unassigned_meas, sensor)` đúng lượt: LiDAR rỗng phải xử lý miss trong FOV và xét xóa; camera rỗng không thay đổi lifecycle. Chỉ LiDAR tạo track, cộng/trừ score và quyết định xóa. Camera chỉ tinh chỉnh `x, P`, vì trong lab nguồn đo camera bổ sung không được dùng làm bằng chứng tồn tại độc lập. `manager.handle_updated_track(track, sensor)` do platform chỉ ghi nhận hit khi sensor là LiDAR.

6. **Điều kiện xác nhận, giữ confirmed sau miss và xóa track?**
   Trong [track_management.py](workspace/track_management.py), track mới có vận tốc 0, vị trí từ phép biến đổi `sens_to_veh`, covariance vị trí `R_rotation * R_measurement * R_rotation^T`, covariance vận tốc từ bình phương `sigma_p44/55/66`, state `initialized`, score `1/window`. LiDAR hit tăng `1/window`, chặn trên ở 1; miss trong FOV giảm `1/window`. Xác nhận khi score **lớn hơn** `confirmed_threshold`; hit chưa đạt ngưỡng chuyển sang `tentative`. Track đã confirmed không bị hạ state khi miss. Xóa nếu `Pxx > max_P` hoặc `Pyy > max_P`, hoặc confirmed có score **nhỏ hơn** `delete_threshold`, hoặc track chưa confirmed có score **≤ 0**. Miss ngoài FOV không giảm score; lượt camera không khởi tạo, đổi score hoặc xét xóa.

## Bonus (không bắt buộc)

- Không. Ưu tiên hoàn tất phần bắt buộc CP0–CP6.

## Khai báo sử dụng AI (bắt buộc)

- Công cụ đã dùng (ChatGPT, Copilot, Claude, …): OpenAI Codex (ChatGPT).
- Dùng cho phần nào (hàm, câu hỏi, debug): Đọc yêu cầu, triển khai toàn bộ hàm Part E–H, thiết lập môi trường, chạy kiểm tra và hỗ trợ soạn cả sáu câu giải thích trong báo cáo.
- Cách bạn đã kiểm tra lại (pytest, chạy Waymo, đối chiếu công thức): Codex đã đối chiếu công thức với hướng dẫn và chạy bộ test gốc: 128 passed; gồm EKF số học, Jacobian camera so với sai phân hữu hạn, FOV/độ sâu, gating/greedy, frame rỗng, giới hạn score và lifecycle chỉ theo LiDAR. Chưa kiểm tra tích hợp Waymo do thiếu dataset. Học viên cần tự đọc, hiểu và giải thích lại mã nguồn/báo cáo trước khi nộp và vấn đáp.

## Checklist nộp

- [x] **Part E–H** đã implement; toàn bộ 128 test passed, không còn failed/xfailed.
- [x] Part A–D giữ nguyên.
- [ ] Đã chạy chấm điểm `--fusion compare --seed 0`, frame 0–198.
- [ ] Đã commit đủ 6 file metrics/log do runner sinh, không sửa tay.
- [ ] Báo cáo có đủ số liệu thực tế và đối chiếu hai mode.
- [x] Đã khai báo sử dụng AI.
- [x] Không commit Waymo, weights, paths.yaml, .env hoặc API key.
- [x] Tên repo đã đổi đúng mẫu yêu cầu.
- [ ] `python tools/check_submission.py` báo `KẾT QUẢ: SẴN SÀNG NỘP`.
- [x] Đã push các checkpoint code E–H và phần giải thích báo cáo.
- [ ] Đã nộp link repo và commit hash cuối trên LMS.
