# Báo cáo thực hành PointPillars — Day 13

## Cá nhân và provenance

- Mã nhóm: Cá nhân, không có nhóm
- Họ và tên: Phù Ngô Việt Anh
- MSSV: 2A202602141
- Trạng thái: `executed-by-student` — tự chạy trên máy local bằng Docker Desktop Linux amd64
- Người thực sự chạy: Phù Ngô Việt Anh
- Ngày/giờ chạy: 02/10/2026, khoảng 15:57–15:58 ICT (UTC+7)
- Hệ máy/architecture: Windows host, Docker Linux `amd64`/`x86_64`, Python 3.11.9
- Image tag: `day13-pointpillars:lc-20261001-amd64`
- Image ID: `sha256:e03983bd922ec29890bf547db8de408402efd82583680b62e671c20da2fd2c82`
- Phiên bản repo: `0831856d921609312d42c7582c366e5a311bb7b1`; manifest ghi `working_tree_dirty=true`
- PCD/frame: KITTI demo 000008 đã chuyển đổi, `frame_id=demo`, input SHA256 `3b5ea3da13e2b19149cab6a8d521c2ca55f2df93f026b5a3f8c273ce70645d60`
- Checkpoint: `/opt/PointPillars/pretrained/epoch_160.pth`; SHA256 `482dfcf63b932cc5ccf0124bbdad52aa51aa33becf87d0a39d61c39b377b5b1`
- Phạm vi: front-window; score threshold `0.3`
- Giới hạn chạy: tối đa 4 CPU và 4 GB RAM/container; `smoke.json` có trạng thái `passed`
- Kênh thứ tư/intensity: reflectance nguồn đã bị bỏ trong PCD; adapter dùng kênh hằng/RGB placeholder `0`; `z_ground=0.075 m`

## Ba lượt inference thật

A/B/C là ba lượt trên cùng PCD và cùng checkpoint. Runner chạy tuần tự từ một lệnh. Số hộp và `mean_z` dưới đây lấy trực tiếp từ `summary.csv`.

| Lượt | delta | Pillar XY | Số hộp | mean_z | File JSON/Side/CSV | Quan sát có bằng chứng |
| --- | ---: | ---: | ---: | ---: | --- | --- |
| A | 0 m | 0.16 m | 1 | 0.330 | `output/run-A/boxes-demo-delta-0-voxel-0.16.json`; `output/run-A/side-demo-delta-0-voxel-0.16.png`; `output/run-A/summary.csv` | 1 hộp `vehicles`, score khoảng 0.322, tâm x≈13.15 m, z≈0.330 m. |
| B | 1.73 m | 0.16 m | 13 | 1.034 | `output/run-B/boxes-demo-delta-1.73-voxel-0.16.json`; `output/run-B/side-demo-delta-1.73-voxel-0.16.png`; `output/run-B/summary.csv` | 10 `vehicles`, 2 `pedestrian`, 1 `two-wheels`; các hộp xuất hiện ở nhiều vùng x trong ảnh Side, khoảng x≈3–43 m. |
| C | 1.73 m | 0.32 m | 6 | 1.091 | `output/run-C/boxes-demo-delta-1.73-voxel-0.32.json`; `output/run-C/side-demo-delta-1.73-voxel-0.32.png`; `output/run-C/summary.csv` | 6 hộp đều là `pedestrian`; số lượng và class khác B, các hộp còn lại tập trung ở các vùng khác nhau trong ảnh Side. |

- A/B — chỉ đổi delta: A có 1 hộp, B có 13 hộp. Ảnh Side cho thấy B không chỉ dịch hộp cũ theo z mà là kết quả chạy lại model trên input đã dịch: số hộp, class và vùng xuất hiện thay đổi rõ rệt. Vì A/B dùng front-window và ảnh Side là phép chiếu x-z, chưa đủ cơ sở kết luận hộp nào đúng hơn nếu không kiểm thêm Top/Front/PCD.
- B/C — chỉ đổi pillar: B có 13 hộp, C có 6 hộp. Khi pillar tăng từ 0.16 m lên 0.32 m, số hộp giảm và class thay đổi từ `vehicles`/`pedestrian`/`two-wheels` sang 6 `pedestrian`. Đây là thay đổi biểu diễn đầu vào; không có đủ ground truth để kết luận C tốt hơn chỉ dựa trên số hộp hoặc score.
- Giới hạn ROI và góc Side: front-window không đại diện toàn scene; ảnh Side chiếu các điểm/hộp khác y lên cùng mặt phẳng x-z nên có thể che khuất hoặc chồng đối tượng. Không dùng Side một mình để kết luận miss, class hoặc yaw.
- JSON chưa đủ cơ sở để import: toàn bộ prediction này là KITTI demo cho bài thực hành, không phải Robotaxi. Các file `qc-cases/case-*.json` có `training_only=true`, không được import vào CVAT. Prediction KITTI cũng không được nạp vào job Robotaxi.

## Ca QC có kiểm soát — không import CVAT

Helper tạo ba ca từ prediction thật B (`source_prediction_sha256=c879d3b31423765777c4b0dbe006ef9937b1248270d1d15c08c8fbe4e1c8f124`); đây không phải ba lượt detector và không phải ground truth.

| Ca | Số hộp lệch z / tổng hộp | Lượng lệch | Class/x/y/yaw có đổi? | Dừng batch, kiểm từng hộp hay chưa rõ? | Bằng chứng |
| --- | ---: | ---: | --- | --- | --- |
| `case-correct` | 0/13 | 0 m | Không đổi | Giữ nguyên phép chuyển; không coi đây là nhãn đúng | `output/qc-cases/case-correct.json`, `output/qc-cases/side-correct.png` |
| `case-batch-z` | 13/13 | -1.805 m ở mọi hộp | Không đổi | **Dừng batch**, kiểm tra transform/frame và báo LC; không sửa tay từng hộp | `output/qc-cases/case-batch-z.json`, `output/qc-cases/side-batch-z.png` |
| `case-one-box-z` | 1/13 | -1.805 m ở một hộp | Không đổi | Kiểm từng hộp bằng nhiều view; chưa kết luận lỗi toàn pipeline | `output/qc-cases/case-one-box-z.json`, `output/qc-cases/side-one-box-z.png` |

Helper tạo biến đổi có chủ đích từ prediction B. Với `z_ground=0.075` và `delta=1.73`, lượng chuyển ngược cần cộng là `z_ground + delta = 1.805 m`. Quan hệ dùng để kiểm là:

```text
z_model  = z_source - z_ground - delta
z_source = z_model + z_ground + delta
```

## Nhận xét cá nhân — Phù Ngô Việt Anh

- Vai trò: tự vận hành runner, kiểm cấu hình/JSON, xem ảnh Side, phân tích ca QC và ghi báo cáo.
- Quan sát: thay đổi delta từ A sang B làm output đổi mạnh từ 1 hộp thành 13 hộp; thay đổi pillar từ B sang C làm output giảm còn 6 hộp. Bằng chứng nằm trong ba `summary.csv`, ba ảnh Side và các JSON tương ứng trong `output/run-A`, `output/run-B`, `output/run-C`.
- Diễn giải z: delta được trừ trước inference và phải được cộng lại khi đưa hộp về hệ tọa độ nguồn; không được dịch hộp thủ công sau inference để thay thế phép chuyển frame.
- Quyết định QC: với `case-batch-z`, cả 13 hộp cùng lệch -1.805 m trong khi class/x/y/yaw giữ nguyên, nên dừng batch và kiểm pipeline. Với `case-one-box-z`, chỉ kiểm đối tượng/hộp bị lệch qua nhiều góc nhìn.
- Điều chưa chắc: chưa có ground truth chất lượng cho KITTI demo và chỉ có front-window/ảnh Side trong output; không kết luận cấu hình B hay C tốt hơn chỉ từ số hộp, mean_z hoặc score.

## LC ghi nhận riêng

- Quyền dùng PCD/image và đúng ca: KITTI demo trong Student bundle, chạy local; không dùng Robotaxi/private input.
- Có chạy thật / chỉ phân tích: đã chạy thật trên máy local; không phải `provided-results`.
- Output đủ: `smoke.json` passed; A/B/C và ba ca QC đều đủ JSON/PNG/CSV/manifest theo runner.
- Không đưa ca lỗi vào CVAT: đã tuân thủ; chỉ dùng các ca để luyện nhận diện lỗi pipeline.
- Nhận xét từng thành viên và quyết định dừng pipeline: có một thành viên; đã ghi ở mục Nhận xét cá nhân và bảng QC.
- Đồng ý chuyển sang chỉnh/QC: phần portal/CVAT và feedback đã làm; A/B/C là phần thực hành KITTI riêng.
