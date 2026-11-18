# Báo cáo Day 09: Hà Khánh, 2A202603221

## 1. Kết luận trước

Pipeline trả về `Hold` với mã thoát `10`. Từ `224` ảnh trong `data/raw`, bước nhận dữ liệu giữ `202` ảnh để xuất bản nháp vào `submission/export/clean_v1/` và loại `22` ảnh lỗi ở cửa vào. Sau bước sửa tự động và vòng duyệt mô phỏng `80` ảnh, tập dữ liệu không còn lỗi bắt buộc, nhưng còn `38/202` ảnh (`18.8%`) mang cảnh báo chưa được kiểm tra, cao hơn ngưỡng `10.0%` của cổng `gate`.

## 2. Lỗi và phân việc

File `02_error_list_v1.txt` ghi nhận toàn bộ lỗi trên `224` ảnh gốc, gồm `22` ảnh bị loại ở bước nhận dữ liệu và `202` ảnh đi tiếp vào kiểm tra nhãn.

| Mã lỗi | Số ảnh/dòng | Ai sửa (máy/người) | Vì sao |
|---|---:|---|---|
| `DUP_IMAGE_EXACT` | 8 ảnh (/224 ảnh) | Máy | Trùng byte hoàn toàn qua md5, xóa bản sao ở bước nhận dữ liệu |
| `LEAK_TEST` | 5 ảnh (/224 ảnh) | Máy | Trùng với tập `test` hoặc `gold_val`, phải loại để tránh rò rỉ đánh giá |
| `DUP_IMAGE_NEAR` | 4 ảnh (/224 ảnh) | Máy | Trùng gần đúng trong ngưỡng pHash `<= 10`, xóa bản sao tự động |
| `BAD_SIZE` | 3 ảnh (/224 ảnh) | Máy | Cạnh ngắn dưới `250px`, không đạt độ phân giải tối thiểu |
| `CORRUPT_IMAGE` | 2 ảnh (/224 ảnh) | Máy | File ảnh hỏng (`UnidentifiedImageError`, `OSError`), không đọc được pixel |
| `BOX_OUT_OF_IMAGE` | 8 dòng (/202 ảnh) | Máy | Tọa độ vượt biên `[0, 1]` ngoài `0.002`, máy cắt về trong ảnh (`clip_box`) |
| `DUP_BOX` | 7 dòng (/202 ảnh) | Máy | Box lặp cùng lớp có `IoU > 0.9`, máy xóa dòng đứng sau (`drop_line`) |
| `BAD_CLASS_ID` | 6 dòng (/202 ảnh) | Máy | ID lớp ngoài `0..3` (`-1`, `4`, `7`, `93`), máy xóa dòng (`drop_line`) |
| `MISSING_LABEL_FILE` | 6 ảnh (/202 ảnh) | Người | Ảnh chưa có file `.txt`, máy ghi `needs_label` và không tạo file rỗng |
| `BOX_ZERO` | 4 dòng (/202 ảnh) | Máy | Box có chiều rộng hoặc chiều cao bằng `0`, máy xóa dòng (`drop_line`) |
| `BAD_FORMAT` | 3 dòng (/202 ảnh) | Máy | Dòng thiếu cột hoặc chứa `nan`, máy xóa dòng (`drop_line`) |
| `BAD_CLASS_NAME` | 2 dòng (`data.yaml`) | Máy | Tên lớp `Person` và `Bus` viết hoa sai, máy ghi lại đúng `cfg["classes"]` |
| `POSSIBLE_MISSING` | 53 cảnh báo (/202 ảnh) | Người | Model `yolo26l` thấy vật thể nhưng nhãn thiếu box, cần người xác nhận |
| `POSSIBLE_FALSE_POS` | 29 cảnh báo (/202 ảnh) | Người | Box nhãn không khớp dự đoán nào của model kiểm tra, có thể là box thừa |
| `POSSIBLE_LOOSE_BOX` | 28 cảnh báo (/202 ảnh) | Người | Box nhãn cùng lớp có `IoU < 0.5` so với dự đoán model, cần chỉnh lại biên |
| `POSSIBLE_WRONG_CLASS` | 18 cảnh báo (/202 ảnh) | Người | Vị trí khớp (`IoU >= 0.5`) nhưng khác lớp giữa nhãn và model, cần người phân định |
| `EMPTY_LABEL` | 12 ảnh (/202 ảnh) | Người | File `.txt` rỗng, cần kiểm tra xem là ảnh nền hay thiếu nhãn |
| `BOX_TINY` | 6 dòng (/202 ảnh) | Người | Cạnh ngắn dưới `12px`, cần đối chiếu guideline để giữ hoặc bỏ |
| `CLASS_RARE` | 4 cảnh báo (`<dataset>`) | Máy (gián tiếp) | Phát sinh từ các ID sai `-1`, `4`, `7`, `93` và hết sau khi xóa `BAD_CLASS_ID` |
| `BOX_HUGE` | 3 dòng (/202 ảnh) | Người | Box chiếm hơn `95%` diện tích ảnh, cần xem lại để tránh gán nhầm nền |

Ở vòng QC 1, bốn mã cảnh báo ngữ nghĩa cộng lại thành `128` dòng (`53 + 29 + 28 + 18`), nhưng sang `04_error_list_v2.txt` con số này còn `123` dòng. Chênh lệch `5` dòng nằm ở mã `POSSIBLE_FALSE_POS` (`29` xuống `24`). Khi `autofix` chạy qua, máy xóa `4` dòng `BOX_ZERO` (`img_0050.txt:1`, `img_0066.txt:3`, `img_0092.txt:2`, `img_0185.txt:10`) vốn có `IoU = 0` với mọi dự đoán nên trước đó bị gắn thêm `POSSIBLE_FALSE_POS`. Dòng `img_0108.txt:1` sau khi được cắt gọn phần tràn biên cũng tăng IoU với dự đoán của model từ `0.299` lên `0.454` (vượt mức `0.3`), nên thoát khỏi cảnh báo này.

Một thay đổi khác sau `autofix` là `EMPTY_LABEL` tăng từ `12` lên `15` ảnh. Ba ảnh tăng thêm là `img_0050`, `img_0104` và `img_0194`: chúng vốn có sẵn file `.txt` nhưng mỗi file chỉ chứa một dòng lỗi (`BOX_ZERO` hoặc `BAD_CLASS_ID`), nên sau khi xóa dòng lỗi thì file trở thành `0` dòng. Nhóm này khác với `6` ảnh `MISSING_LABEL_FILE` vốn không có file `.txt` ngay từ đầu (`img_0031`, `img_0053`, `img_0126`, `img_0135`, `img_0170`, `img_0219`).

Hàng đợi `05_review_queue.csv` sắp xếp `125` ảnh theo tổng trọng số trong `cfg["review"]["weights"]` nhân với `1 + conf`. Mức trọng số `NEEDS_LABEL: 10` đẩy toàn bộ `6` ảnh thiếu file nhãn lên các vị trí `2` đến `7`, ngay sau `img_0020` (`11.524` điểm do cộng dồn nhiều lỗi). Nhóm tiếp theo là các ảnh nghi nhầm lớp hoặc thiếu nhãn với độ tin cậy cao, còn các cảnh báo nhẹ như `BOX_TINY` (`0.3`) hay `BOX_HUGE` (`0.5`) nằm ở cuối danh sách. Trong giới hạn `80` ảnh được duyệt, ngoài `6` ảnh `NEEDS_LABEL`, người duyệt xử lý `74` ảnh mang cảnh báo ngữ nghĩa, bao phủ `94/123` dòng cảnh báo ngữ nghĩa của vòng QC 2 (`18/18` `POSSIBLE_WRONG_CLASS`, `51/53` `POSSIBLE_MISSING`, `13/24` `POSSIBLE_FALSE_POS` và `12/28` `POSSIBLE_LOOSE_BOX`).

## 3. Calibration (2 ngưỡng)

Các chỉ số Precision và Recall dưới đây được đo trên `39` ảnh của tập `data/calib` dựa trên đáp án lỗi chèn sẵn trong `data/calib/MANIFEST.csv` (so theo cặp `(ảnh, dòng)` với lỗi cú pháp và theo `ảnh` với lỗi trùng hoặc ngữ nghĩa). Tập `data/raw` (`202` ảnh qua cửa vào) không có đáp án chuẩn để tính các chỉ số này.

| Tham số | Khởi đầu | Đã chọn | Precision / Recall trên `calib` (trước sang sau) | Lý do (đánh đổi) |
|---|---|---|---|---|
| `duplicates.phash_dist` | 2 | 10 | `DUP_IMAGE_NEAR`: 1.00 / 0.50 sang 1.00 / 0.67; `DUP_IMAGE_EXACT`: 1.00 / 1.00 sang 1.00 / 1.00 | Trùng ảnh dẫn tới xóa ảnh tự động nên cần giữ precision `1.00` để không xóa nhầm ảnh đúng; mức `10` tăng recall từ `0.50` lên `0.67` mà chưa tạo báo động giả |
| `second_opinion.conf` | 0.25 | 0.6 | `POSSIBLE_WRONG_CLASS`: 0.56 / 0.83 sang 0.62 / 0.83; `POSSIBLE_MISSING`: 0.50 / 1.00 sang 0.57 / 1.00 | Nâng `conf` từ `0.25` lên `0.6` lọc bớt các dự đoán nhiễu của model (precision tăng ở cả hai nhóm lỗi chính) trong khi vẫn giữ recall `1.00` cho `MISSING`, tránh hiện tượng tụt recall xuống `0.88` ở `0.7` và `0.75` ở `0.8` |

Nếu chỉ nhìn bảng sweep trên `39` ảnh của `data/calib`, ngưỡng `phash_dist = 16` trông hợp lý nhất vì đưa recall của `DUP_IMAGE_NEAR` lên `0.83` (`precision = 1.00`, `F1 = 0.91`). Bảng sweep không đo `LEAK_TEST` do `data/calib/MANIFEST.csv` không có mẫu rò rỉ. Trong `pipeline/checks/duplicates.py`, tham số `phash_dist` lại được dùng chung cho cả việc đối chiếu với `140` ảnh kho `test`/`gold_val` (`LEAK_TEST`) lẫn đối chiếu nội bộ (`DUP_IMAGE_NEAR`) theo cùng điều kiện khoảng cách Hamming `<= phash_dist`. Khi nâng ngưỡng từ `10` lên `16` trên `data/raw`, tổng số ảnh bị loại vì trùng nhảy từ `17` lên `26`: phát sinh thêm `3` ca `DUP_IMAGE_NEAR` (`img_0011`, `img_0084`, `img_0177`) và `6` ca `LEAK_TEST` (`img_0013`, `img_0017`, `img_0021`, `img_0033`, `img_0038`, `img_0115`). Cả `9` ca phát sinh thêm đều có khoảng cách Hamming đúng bằng `16` và bước nhận dữ liệu không có thay đổi nào khác. Lúc mở xem trực tiếp các cặp `DUP_IMAGE_NEAR`, `img_0008.jpg` (hai người trượt tuyết) bị ghép nhầm với `img_0011.jpg` (người chơi tennis), còn `img_0113.jpg` (người ngồi bàn) bị ghép nhầm với `img_0177.jpg` (người lướt sóng). Tôi giữ `phash_dist = 10` để tránh xóa nhầm ảnh khác nội dung ở khoảng cách `16`. Mức `10` vẫn bắt được các cặp trùng thật như `img_0077.jpg`/`img_0223.jpg` (khoảng cách `6`) và `img_0061.jpg`/`img_0186.jpg` (khoảng cách `10`).

## 4. Cửa chặn và điều pipeline KHÔNG bắt được

(a) Thử nghiệm với ngân sách bằng 0 cho thấy cổng `gate` chặn xuất bản đúng quy tắc. Chạy lệnh `python -m pipeline.run --budget 0 --reports work/rep_nb --export-dir work/exp_nb` ghi nhận `reviewed = 0`. Sáu ảnh thiếu file nhãn (`MISSING_LABEL_FILE`) không có người gán nên vẫn nằm lại ở vòng QC 3 (`n_errors = 6`, `n_warn_images = 112/202`). Kết quả trả về là `Rework` kèm mã thoát `20`. File `work/rep_nb/08_summary.json` ghi `exported: null` và thư mục `work/exp_nb` không tồn tại.

(b) Qua đối chiếu trực tiếp dữ liệu trên `work/run/ds`, pipeline vẫn còn các điểm mù sau:

1. Ảnh trùng khi cắt xén và đổi kích thước mạnh: hai ảnh `img_0074.jpg` và `img_0084.jpg` chụp cùng người phụ nữ cầm ô trắng nhưng khung cắt khác phần rìa. Khoảng cách pHash giữa chúng lên tới `16`, vượt ngưỡng `phash_dist = 10` nên cặp này lọt qua bước nhận dữ liệu.
2. Box lỏng ở mức trung bình: do `iou_loose = 0.5` cố định, các box có IoU từ `0.50` đến dưới `0.65` so với dự đoán độ tin cậy cao đều không bị báo (`4` trường hợp trong `work/run/ds`). Trên `img_0190`, nhãn `person` có kích thước `(0.250, 0.356)` rộng gần gấp đôi dự đoán của `yolo26l` (`conf = 0.91`, kích thước `(0.191, 0.241)`). Dù vậy, phép đo `IoU = 0.516` đạt ngưỡng `0.5` nên pipeline giữ im lặng.
3. Vật thể nhỏ bị lọc trước khi so sánh: do `min_area_px = 600` px², check `second_opinion` bỏ qua `44` dự đoán có `conf >= 0.6` trên `202` ảnh đạt chuẩn. Trong `44` dự đoán này, `16` dự đoán đã khớp với box nhãn sẵn có (`IoU >= 0.3`). Ở `28` dự đoán còn lại chưa khớp box nào (`IoU < 0.3`), có `16` dự đoán mang cạnh ngắn dưới `12px` (thuộc nhóm `BOX_TINY`, có thể chủ đích không gán theo guideline). Chỉ còn tối đa `12` dự đoán đạt cạnh ngắn từ `12px` trở lên và nằm dưới ngưỡng `600 px²`; phần này vẫn có khả năng lẫn dự đoán sai của `yolo26l` cần người xem lại. Một ví dụ đáng nghi trên `img_0007` (`640 x 427`) là hai xe ô tô nhỏ có `conf = 0.85` (`30.9 x 15.3 px`, `472.3 px²`) và `conf = 0.65` (`26.1 x 14.2 px`, `371.2 px²`) ở các vị trí chưa có nhãn trong file `.txt`.

## 5. Train: raw vs clean (tùy chọn, chấm riêng)

Chưa chạy. Lý do: chưa thực hiện bước huấn luyện trên Google Colab (`--no-train`).

## 6. Quyết định và ghi chú AI

(a) Dữ liệu hiện dừng ở trạng thái phát hành nháp để thử nghiệm nội bộ, chưa thể đưa ra phát hành chính thức. Với `202` ảnh đạt bước nhận dữ liệu, ngưỡng `Hold` `10.0%` cho phép tối đa `floor(202 * 0.10) = 20` ảnh mang cảnh báo chưa kiểm tra (`20/202 = 9.9%`). Từ mức `38` ảnh hiện tại, cần duyệt thêm ít nhất `18` ảnh để đạt điều kiện `Release`. Kiểm tra trên `05_review_queue.csv` cho thấy toàn bộ `18` ảnh từ vị trí `81` đến `98` (`img_0210` đến `img_0142`, điểm ưu tiên `3.333` xuống `2.487`) đều đang mang cảnh báo `WARN` (`POSSIBLE_MISSING`, `POSSIBLE_LOOSE_BOX`, `BOX_HUGE`), còn các ảnh chỉ có nhãn `FIXED_*` (không còn cảnh báo) bắt đầu từ vị trí `100` trở xuống. Chạy thử lại `simulate` và `gate.decide` với `budget = 98` xác nhận số ảnh cảnh báo chưa duyệt giảm đúng từ `38` xuống `20` (`warn_ratio = 0.099 < 0.10`, `n_errors = 0`) và `gate` trả về `Release`. Khi tăng ngân sách duyệt, nên ưu tiên xử lý `3` dòng `POSSIBLE_WRONG_CLASS` và `13` dòng `POSSIBLE_MISSING` còn lại trong `07_error_list_v3.txt`.

(b) Ghi chú dùng AI:

Prompt chính gửi kèm docstring của 4 module (`label_format.py`, `gate.py`, `autofix.py`, `second_opinion.py`), định dạng 5 cột của nhãn YOLO, bảng mã lỗi bắt buộc/cảnh báo và yêu cầu chỉ chỉnh 2 ngưỡng `# <-- CALIBRATE` trong `configs/pipeline.yaml`, giữ nguyên `data/raw` và các file đề.

Sau khi có code, tôi kiểm tra bằng toàn bộ `79` unit test và e2e test qua `pytest`, chạy sweep đo precision/recall trên `data/calib`, rồi đối chiếu số liệu trong `submission/reports/` với ảnh gốc.

Trong lúc kiểm thử, đoạn script gọi `get_preds(raw, cfg, "checker")` trực tiếp trên `data/raw` từng văng `RuntimeError` do tập thô chứa ảnh lỗi chưa có trong `checker_preds.json` khi máy chưa cài `ultralytics`, buộc phải chuyển sang đọc từ `load_cache` hoặc chỉ gọi sau bước nhận dữ liệu. Việc mở ảnh đối chiếu thủ công ở mức `phash_dist = 16` cũng giúp phát hiện cặp `img_0074` và `img_0084` thực chất là trùng cắt ghép, tránh đánh giá nhầm với các cặp sai nội dung như `img_0008`/`img_0011` và `img_0113`/`img_0177`.
