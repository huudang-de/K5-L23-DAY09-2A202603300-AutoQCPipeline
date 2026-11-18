# Báo cáo Day 09 — Nguyễn Hữu Đăng, 2A202603300

## 1. Kết luận trước
Pipeline kết luận **HOLD** (mã thoát `10`). Trong **224** ảnh đầu vào (`data/raw`), bước cửa vào loại **22** ảnh (hỏng, quá nhỏ, trùng, rò rỉ), còn **202** ảnh đi qua QC và được xuất bản nháp vào `submission/export/clean_v1`. Lý do chưa thể Release: tuy không còn lỗi bắt buộc (`n_errors = 0`), tỉ lệ ảnh còn cảnh báo chưa được người duyệt là **38/202 (18.81%) >= ngưỡng 10.00%** do hàng đợi 125 ảnh vượt quá ngân sách duyệt 80 ảnh.

## 2. Lỗi và phân việc
Số liệu lấy từ `submission/reports/02_error_list_v1.csv` (trên 224 ảnh gốc):

| Mã lỗi | Số dòng / ảnh | Ai sửa (máy/người) | Vì sao |
|---|---:|---|---|
| `CORRUPT_IMAGE` | 2 dòng / 2 ảnh | Máy (`remove_image`) | Ảnh hỏng không giải mã được, không thể dùng train |
| `BAD_SIZE` | 3 dòng / 3 ảnh | Máy (`remove_image`) | Cạnh ngắn < 250 px, vi phạm quy tắc đầu vào |
| `LEAK_TEST` | 5 dòng / 5 ảnh | Máy (`remove_image`) | Trùng kho `test`/`gold_val`, bắt buộc loại để tránh rò rỉ |
| `DUP_IMAGE_EXACT` | 8 dòng / 8 ảnh | Máy (`remove_image`) | Trùng từng byte (md5), giữ bản đầu và loại bản lặp |
| `DUP_IMAGE_NEAR` | 4 dòng / 4 ảnh | Máy (`remove_image`) | Trùng gần đúng (phash <= 10), loại bản lặp |
| `BAD_CLASS_NAME` | 2 dòng / 1 (`data.yaml`) | Máy (`rename_classes`) | Sai tên/thứ tự lớp trong `data.yaml`, chuẩn hoá theo `cfg["classes"]` |
| `MISSING_LABEL_FILE` | 6 dòng / 6 ảnh | Người (`needs_label`) | Máy không được tự bịa nhãn; chuyển ưu tiên cao nhất cho người gán |
| `BAD_FORMAT` | 3 dòng / 3 ảnh | Máy (`drop_line`) | Dòng nhãn thiếu cột / `nan` / sai kiểu số, không khôi phục được |
| `BAD_CLASS_ID` | 6 dòng / 6 ảnh | Máy (`drop_line`) | ID lớp ngoài `0..3`, xoá dòng sai cú pháp |
| `BOX_ZERO` | 4 dòng / 4 ảnh | Máy (`drop_line`) | Box có `w <= 0` hoặc `h <= 0`, vô nghĩa về hình học |
| `BOX_OUT_OF_IMAGE` | 8 dòng / 8 ảnh | Máy (`clip_box`) | Cắt toạ độ box về `[0, 1]` (hoặc xoá nếu nằm hẳn ngoài ảnh) |
| `DUP_BOX` | 7 dòng / 7 ảnh | Máy (`drop_line`) | Hai box cùng lớp chồng lấn > 0.9 IoU, xoá dòng sau |
| `EMPTY_LABEL` | 12 dòng / 12 ảnh | Người (duyệt) | Cần người xác nhận là ảnh nền thật hay bị xoá nhầm hết nhãn |
| `CLASS_RARE` | 4 dòng / 1 (`<dataset>`) | Theo dõi thống kê | Cảnh báo cấp dataset để kiểm tra mất cân bằng lớp |
| `BOX_TINY` | 6 dòng / 4 ảnh | Người (duyệt) | Cạnh ngắn < 12 px, người xem lại để tránh nhiễu nhãn quá nhỏ |
| `BOX_HUGE` | 3 dòng / 3 ảnh | Người (duyệt) | Box chiếm > 95% diện tích ảnh, dễ là box phình/nhầm |
| `POSSIBLE_MISSING` | 53 dòng / 52 ảnh | Người (duyệt) | Model `yolo26l` thấy vật nhưng thiếu nhãn; model có thể sai nên cần người |
| `POSSIBLE_FALSE_POS` | 29 dòng / 27 ảnh | Người (duyệt) | Nhãn không khớp dự đoán nào của model; cần người xác nhận |
| `POSSIBLE_LOOSE_BOX` | 28 dòng / 24 ảnh | Người (duyệt) | Đúng vật nhưng IoU thấp (< 0.5), cần người chỉnh lại viền box |
| `POSSIBLE_WRONG_CLASS` | 18 dòng / 18 ảnh | Người (duyệt) | Model nghi nhầm lớp hợp lệ (car/truck/bus/person), cần người phân xử |

`05_review_queue.csv` (125 ảnh) xếp ưu tiên theo tổng điểm trọng số rủi ro trong `configs/pipeline.yaml`: ảnh thiếu file nhãn (`NEEDS_LABEL`: 10 điểm) và ảnh chồng nhiều lỗi ngữ nghĩa nặng (như `img_0020` có `FIXED_DROP_LINE;POSSIBLE_MISSING;POSSIBLE_WRONG_CLASS`) đứng đầu hàng đợi, còn ảnh chỉ có `BOX_TINY` (0.3) hoặc `BOX_HUGE` (0.5) đứng cuối (`img_0196`, `img_0026`, `img_0051`). Với ngân sách 80 ảnh (`06_review_log.csv`), người duyệt xử lý hết toàn bộ 6 ảnh `MISSING_LABEL_FILE` và 74 ảnh cảnh báo ưu tiên cao nhất; 45 ảnh còn lại ở cuối hàng đợi chưa được duyệt (trong đó 38 ảnh mang cảnh báo chưa duyệt tạo nên tỉ lệ Hold 18.81%).

## 3. Calibration (2 ngưỡng)
| Tham số | Khởi đầu | Đã chọn | Precision / Recall (trước -> sau) | Lý do (đánh đổi) |
|---|---:|---:|---|---|
| `duplicates.phash_dist` | 2 | 10 | `DUP_IMAGE_NEAR`: `1.00 / 0.50` -> `1.00 / 0.67` (`F1: 0.67 -> 0.80`) | Lỗi xoá ảnh tự động cần precision tuyệt đối `1.00`; chọn `10` để bắt thêm cặp trùng thật trên `raw` (`img_0186` vs `img_0061` ở `dist=10`, `img_0223` vs `img_0077` ở `dist=6`) mà không xoá nhầm ảnh khác nhau |
| `second_opinion.conf` | 0.25 | 0.6 | `WRONG_CLASS`: `0.56 / 0.83 -> 0.62 / 0.83`; `MISSING`: `0.50 / 1.00 -> 0.57 / 1.00` | Tăng `conf` lên `0.6` giúp cắt bớt báo động giả cho hàng đợi duyệt trong khi vẫn giữ nguyên recall `1.00` cho `MISSING` và `0.83` cho `WRONG_CLASS` (nếu tăng lên `0.7–0.8`, recall của `MISSING` giảm xuống `0.88–0.75`) |

**Một chỗ calibration có thể đánh lừa bạn:** Trên bộ `data/calib` (39 ảnh), `phash_dist=16` đạt F1 cao nhất (`0.91`, precision `1.00`, recall `0.83`). Tuy nhiên khi kiểm tra thực tế trên `data/raw` (224 ảnh), nâng từ `12` lên `16` làm `LEAK_TEST` tăng vọt từ 5 lên 11 ảnh và `DUP_IMAGE_NEAR` tăng từ 4 lên 7 ảnh; mở xem cặp bị cờ ở `dist=16` như `img_0008.jpg` (hai người trượt tuyết) và `img_0011.jpg` (người chơi tennis) cho thấy hai ảnh hoàn toàn khác nhau. Nếu chỉ tối ưu theo F1 trên `calib`, pipeline sẽ xoá nhầm nhiều ảnh sạch trên `raw`.

## 4. Cửa chặn và điều pipeline KHÔNG bắt được
- **(a) Chứng minh gate chặn đúng:** Khi chạy `python -m pipeline.run --budget 0 --reports work/rep_nb --export-dir work/exp_nb` (ngân sách người duyệt bằng 0 nên 6 ảnh `MISSING_LABEL_FILE` vẫn còn nguyên sau vòng 3), pipeline kết luận **REWORK**, trả mã thoát **`20`**, và **không tạo** thư mục xuất `work/exp_nb`.
- **(b) >= 2 điểm mù của pipeline:**
  1. *Lỗi mà cả nhãn và `yolo26l` cùng bỏ sót hoặc cùng gán sai:* khi nhìn trực quan `work/viz/img_0001.png`, người lính đứng bên phải-giữa bị thiếu box; hoặc các vật có diện tích < `min_area_px` (600 px²) bị `second_opinion` bỏ qua nên lỗi thiếu nhãn ở vật nhỏ vẫn lọt qua nếu không mở kiểm tra mắt thường.
  2. *Ảnh trùng biến đổi mạnh hoặc cắt xén lớn:* với `phash_dist=10`, các ảnh trùng bị cắt mạnh hoặc xoay/đổi bố cục có khoảng cách Hamming > 10 vẫn lọt qua bước `duplicates` (trên `calib` recall chỉ đạt 0.67).

## 5. Train: raw vs clean (tùy chọn, chấm riêng)
Chưa chạy (tập trung hoàn thiện toàn bộ phần lõi QC pipeline, calibration và báo cáo trước; thí nghiệm train so sánh `raw` vs `clean` trên Google Colab sẽ được bổ sung sau).

## 6. Quyết định và ghi chú AI
- **(a) Quyết định cuối:** **HOLD**. Để chuyển sang **Release**, cần bổ sung ngân sách duyệt người cho ít nhất 18 trong số 38 ảnh còn cảnh báo chưa duyệt (đưa số ảnh cảnh báo chưa duyệt xuống <= 20/202, tức `< 10.00%`).
- **(b) Ghi chú AI:**
  - *Prompt chính:* Yêu cầu triển khai từng module theo đúng docstring, bảng mã lỗi, ngưỡng trong `configs/pipeline.yaml` và các ca biên của đề bài (`docs/PROMPTING.md`).
  - *Cách kiểm code AI:* Đọc kiểm tra từng dòng logic, chạy đầy đủ 79 unit test + e2e (`python -m pytest -q`), chạy `pipeline.run` trên `data/raw` và đối chiếu trực quan bằng `viz_labels.py`.
  - *1 lần AI sai:* Khi chạy câu lệnh kiểm tra nhanh `duplicates` trên `data/raw` qua PowerShell, lệnh inline dùng dấu ngoặc kép lồng nhau bị PowerShell nuốt ký tự thoát dẫn tới `SyntaxError`; đã sửa bằng cách dùng PowerShell here-string `@' ... '@` và kiểm chứng lại kết quả.
