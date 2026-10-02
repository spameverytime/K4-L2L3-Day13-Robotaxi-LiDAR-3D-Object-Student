# Báo cáo thực hành PointPillars — Day 13

Giữ bản đã điền ngoài Git, trong thư mục nhóm private do LC thu. Đây là kiểm tra formative; không ghi điểm của người khác.

## Nhóm và provenance

- Mã nhóm/phòng: Nhóm thực hành Day 13 (Phòng Lab)
- Thành viên: xem `TEAMMATES.md` (họ tên/MSSV, vai trò từng lượt).
- Trạng thái: `executed-by-group`
- Người thực sự chạy; ngày/giờ; hệ máy/architecture: dp; 2026-10-02 12:20 (UTC+7); Linux 6.8.0 / x86_64 (amd64)
- Image tag và image ID; phiên bản repo:
  - Image tag: `day13-pointpillars:lc-20261001-amd64`
  - Image ID: `sha256:e03983bd922ec29890bf547db8de408402efd82583680b62e671c20da2fd2c82`
  - Phiên bản repo: commit `e226b934c656f23c0da70b1e12cbf365fb78dd82`
- PCD được cấp / frame_id; nơi được phép chạy; fingerprint nếu LC cấp:
  - File: `data/demo.pcd` (frame_id: `demo`, 17238 điểm)
  - Fingerprint SHA256: `3b5ea3da13e2b19149cab6a8d521c2ca55f2df93f026b5a3f8c273ce70645d60`
  - Giấy phép: CC-BY-NC-SA-3.0
- Checkpoint: PointPillars KITTI có sẵn trong image; ghi checkpoint ID/hash nếu LC cấp:
  - Đường dẫn: `/opt/PointPillars/pretrained/epoch_160.pth`
  - SHA256: `482dfcf63b932cc5ccf012b4bbdad52aa51aa33becf87d0a39d61c39b377b5b1`
- Phạm vi: front-window; score threshold: 0.3
- Giả định kênh thứ tư/intensity và nguồn z_ground:
  - Quét lưu `rgb`, không có LiDAR reflectance thật (được gán placeholder 0). Model adapter đọc đám mây 2 lượt: lượt reflectance 0 nhận diện `vehicles`, lượt 0.7 nhận diện `pedestrian` và `two-wheels`, sau đó lọc trùng.
  - Nguồn z_ground: `0.075 m` (ước lượng mặt đất từ phân bố z của đám mây điểm).

## Ba lượt inference thật

A/B/C là ba lượt trên cùng PCD. Runner chạy đủ ba lượt từ một lệnh. Lấy **Số hộp** từ `n_boxes`, **mean_z** từ `mean_z` trong `run-A/B/C/summary.csv`; không tự tính lại hoặc đoán. `mean_z` không phải điểm chất lượng. Mở `side-*.png`, đối chiếu `boxes-*.json` để ghi quan sát. Số hộp không phải đáp án cần khớp nhóm khác.

| Lượt | delta | Pillar XY | Số hộp | mean_z | File JSON/Side/CSV | Quan sát có bằng chứng |
| --- | --- | --- | --- | --- | --- | --- |
| A | 0 | 0.16 | 1 | 0.330 | `run-A/summary.csv`, `run-A/boxes-demo-delta-0-voxel-0.16.json` | Phát hiện duy nhất 1 xe ở x=13.15m (score 0.322); trên `side-demo-delta-0-voxel-0.16.png`, đáy hộp chìm xuống z≈-0.40m dưới mặt đường cục bộ. |
| B | 1.73 | 0.16 | 13 | 1.034 | `run-B/summary.csv`, `run-B/boxes-demo-delta-1.73-voxel-0.16.json` | Phát hiện 13 hộp (10 vehicles, 2 pedestrian, 1 two-wheels); trên `side-demo-delta-1.73-voxel-0.16.png`, đáy hộp bám sát mặt đường cục bộ; score xe đạt 0.933. |
| C | 1.73 | 0.32 | 6 | 1.091 | `run-C/summary.csv`, `run-C/boxes-demo-delta-1.73-voxel-0.32.json` | Phát hiện 6 hộp nhưng 100% bị gán nhãn `pedestrian`, mất hoàn toàn 10 xe ô tô; trên `side-demo-delta-1.73-voxel-0.32.png`, các hộp thu hẹp thành các cọc đứng hẹp do ô pillar 0.32m làm giảm độ phân giải không gian và không khớp với anchor pretrained. |

- A/B — chỉ đổi delta: A có 1 hộp; B có 13 hộp. Ảnh `side-demo-delta-0-voxel-0.16.png` và `side-demo-delta-1.73-voxel-0.16.png` khác ở vùng x≈13m: ở A đáy hộp bị chìm sâu dưới mặt đất (z≈-0.40m) và bỏ sót hầu hết các xe; ở B các hộp bám sát mặt đường cục bộ theo độ dốc địa hình và phát hiện thêm 12 hộp (gồm cả người đi bộ và xe hai bánh). Đây là chạy lại model trên input khác, không chỉ dịch hộp cũ; điều em còn chưa chắc là các xe ở xa (x>50m) có đủ điểm phản xạ LiDAR để xác định chính xác kích thước và góc xoay hay không.
- B/C — chỉ đổi pillar: B có 13 hộp; C có 6 hộp. File `boxes-demo-delta-1.73-voxel-0.32.json` khác ở chỗ: toàn bộ 10 xe ở B biến mất, thay vào đó C dự đoán 6 hộp `pedestrian` tại vị trí các cụm điểm xe cũ. Số lượng và class thay đổi hoàn toàn do kích thước pillar tăng gấp đôi làm co tỷ lệ không gian feature map so với anchor box của model. Không có đủ bằng chứng để kết luận C tốt hơn; thực tế C kém hơn nhiều do phá vỡ biểu diễn hình học và phân loại sai toàn bộ xe.
- Giới hạn ROI và góc Side ảnh hưởng cách đọc miss/yaw thế nào?
  - ROI front-window giới hạn trường nhìn phía trước xe tự hành, không quét vùng phía sau (rear rectangle).
  - Góc nhìn Side chiếu lên mặt phẳng x-z, giúp quan sát tốt chiều cao và đáy hộp so với mặt đường, nhưng không phân biệt được vị trí y (trái/phải) và hoàn toàn không thể kiểm tra góc yaw (đầu/đuôi xe), dễ dẫn đến sai lệch hướng 180° nếu không kết hợp với góc Trên (Top-view) và ảnh camera.
- JSON nào còn chưa đủ cơ sở để import? Cần kiểm gì tiếp?
  - Cả 3 file JSON (`run-A`, `run-B`, `run-C`) đều chưa đủ cơ sở để import vào CVAT:
    - `run-A`: Thiếu hộp trầm trọng và đáy hộp bị chìm dưới đất.
    - `run-C`: Lỗi phân loại nghiêm trọng (nhận nhầm xe thành người đi bộ).
    - `run-B`: Dù là baseline tốt nhất nhưng vẫn chỉ là pre-label từ mô hình pretrained trên KITTI, chưa có nhãn `Animal` và `Obstacle` theo schema Robotaxi, chưa được kiểm chứng yaw 180° và các đối tượng bị che khuất. Cần kiểm tra thủ công qua 4 góc nhìn và ảnh camera trước khi chấp nhận.

## Ca QC có kiểm soát — không import CVAT

| Ca | Số hộp lệch z / tổng hộp | Lượng lệch | Class/x/y/yaw có đổi? | Dừng batch, kiểm từng hộp hay chưa rõ? | Bằng chứng |
| --- | --- | --- | --- | --- | --- |
| case-correct | 0 / 13 | 0 m | Không đổi | Giữ nguyên / chấp nhận để rà soát tiếp | `side-correct.png`: 13 hộp bám sát mặt đường cục bộ theo địa hình dốc. |
| case-batch-z | 13 / 13 | -1.805 m | Không đổi | **Dừng batch, kiểm tra pipeline** | `side-batch-z.png`: Toàn bộ 13/13 hộp đồng loạt chìm xuống dưới mặt đất 1.805m do thiếu bước cộng ngược `delta + z_ground` trong pipeline chuyển đổi hệ tọa độ. |
| case-one-box-z | 1 / 13 | -1.805 m | Không đổi | **Kiểm tra/chỉnh sửa từng hộp** | `side-one-box-z.png`: 12 hộp vẫn bám đường chuẩn, chỉ duy nhất hộp đầu tiên ở x≈8m bị chìm do lỗi ước lượng mặt đất cục bộ quanh đối tượng đó. |

Ghi rõ helper tạo biến đổi có chủ đích từ prediction, không phải kết quả inference riêng hoặc nhãn đúng.

## Nhận xét cá nhân

- **Học viên: Phạm Xuân Duy (MSSV: 2A202602093) — Luân chuyển đảm nhiệm các vai trò trong thực hành**:
  - **Vai trò đã làm**: Trực tiếp vận hành runner/Docker container trên hệ máy Linux x86_64; kiểm tra cấu hình tham số và dữ liệu JSON; phân tích hình học qua ảnh Side chiếu x-z; ghi chép nhật ký thực nghiệm và tổng hợp báo cáo.
  - **Quan sát A/B/C có dẫn chứng file/vùng**:
    - Đối chiếu A/B (`summary.csv` và `boxes-*.json`): Khi tăng `delta` từ 0m lên 1.73m, số hộp tăng vọt từ 1 lên 13, `mean_z` tăng từ 0.330m lên 1.034m. Trên `side-demo-delta-0-voxel-0.16.png`, hộp xe ở A bị chìm xuống dưới mặt đất cục bộ ($z_{\text{bottom}} \approx -0.40\text{ m}$), trong khi ở `side-demo-delta-1.73-voxel-0.16.png` các hộp bám sát mặt đường và thích ứng với độ dốc địa hình.
    - Đối chiếu B/C (`boxes-demo-delta-1.73-voxel-0.32.json`): Tăng `voxel_size` từ 0.16m lên 0.32m làm mất toàn bộ 10 xe ô tô, chỉ còn lại 6 hộp bị gán nhãn `pedestrian` với chiều dài ngắn ($0.58 - 1.07\text{ m}$) do ô pillar to gấp đôi làm co tỷ lệ đặc trưng không gian so với anchor box của model pretrained.
  - **Diễn giải phép z thuận/ngược**:
    - Phép z thuận: $z_{\text{model}} = z_{\text{source}} - z_{\text{ground}} - \text{delta}$ nhằm chuẩn hóa cao độ đám mây điểm về hệ quy chiếu mặt đất mà PointPillars đã học trên KITTI.
    - Phép z ngược: $z_{\text{source}} = z_{\text{model}} + \text{delta} + z_{\text{ground}}$ nhằm đưa hộp dự đoán trở lại hệ tọa độ cảm biến nguồn. Thiếu bước này sẽ gây lỗi dịch toàn bộ batch chìm xuống lòng đất như trong `case-batch-z`.
    - Phép dịch delta trước inference làm thay đổi phân bố điểm đưa vào mạng nơ-ron, hoàn toàn khác với việc tịnh tiến tọa độ hộp sau khi model đã xuất kết quả.
  - **Quyết định lỗi batch và hành động**:
    - Với lỗi toàn batch (`case-batch-z` lệch 1.805m trên 13/13 hộp): Hành động là **dừng toàn bộ pipeline** để rà soát code chuyển đổi tọa độ; không tốn công sửa tay từng hộp.
    - Với lỗi cục bộ (`case-one-box-z` chỉ lệch 1 hộp ở x≈8m): Hành động là **kiểm tra và điều chỉnh từng đối tượng** trên CVAT vì 12 hộp còn lại đều khớp chuẩn.
  - **Điều chưa chắc**:
    - Ở góc nhìn Side (hình chiếu x-z), ta không thể xác định hướng yaw (đầu/đuôi xe), cần đối chiếu thêm góc Trên (Top-view) và ảnh camera để tránh lỗi xoay ngược 180°.
    - Các đối tượng ở khoảng cách xa ($x > 50\text{ m}$) có mật độ điểm LiDAR rất thưa thớt, cần thận trọng khi xác định ranh giới kích thước hộp.

## LC ghi nhận riêng

- Quyền dùng PCD/image và đúng ca:
- Có chạy thật / chỉ phân tích; còn cần lượt thực hành bổ sung:
- Output đủ, giữ bản gốc, không đưa ca lỗi vào CVAT:
- Nhận xét từng thành viên và quyết định dừng pipeline:
- Đồng ý chuyển sang chỉnh/QC / cần bổ sung; lý do:
