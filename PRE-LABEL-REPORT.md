# Báo cáo thực hành PointPillars — Day 13

Giữ bản đã điền ngoài Git, trong thư mục nhóm private do LC thu. Đây là kiểm tra formative; không ghi điểm của người khác.

## Nhóm và provenance

- Mã nhóm/phòng:
- Thành viên: xem `TEAMMATES.md` (Võ Thành Đức/2A202602167, vai trò từng lượt).
- Trạng thái: `executed-by-group` / `executed-on-room-LC-machine` / `provided-results`.
- Người thực sự chạy; ngày/giờ; hệ máy/architecture:
- Image tag và image ID; phiên bản repo:
- PCD được cấp / frame_id; nơi được phép chạy; fingerprint nếu LC cấp:
- Checkpoint: PointPillars KITTI có sẵn trong image; ghi checkpoint ID/hash nếu LC cấp:
- Phạm vi: front-window; score threshold:
- Giả định kênh thứ tư/intensity và nguồn z_ground:

## Ba lượt inference thật

A/B/C là ba lượt trên cùng PCD. Runner chạy đủ ba lượt từ một lệnh. Lấy **Số hộp** từ `n_boxes`, **mean_z** từ `mean_z` trong `run-A/B/C/summary.csv`; không tự tính lại hoặc đoán. `mean_z` không phải điểm chất lượng. Mở `side-*.png`, đối chiếu `boxes-*.json` để ghi quan sát. Số hộp không phải đáp án cần khớp nhóm khác.

| Lượt | delta | Pillar XY | Số hộp | mean_z | File JSON/Side/CSV                                            | Quan sát có bằng chứng                                                                                                               |
| ---- | ----- | --------- | ------ | ------ | ------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------ |
| A    | 0     | 0.16      | 1      | 0.330  | run-A/boxes-demo.json, run-A/side-demo.png, run-A/summary.csv | Chỉ nhận diện được 1 vehicle duy nhất do point cloud bị lệch cao độ so với mặt đất chuẩn của model KITTI.                            |
| B    | 1.73  | 0.16      | 13     | 1.034  | run-B/boxes-demo.json, run-B/side-demo.png, run-B/summary.csv | Baseline chuẩn: nhận diện 13 hộp (10 vehicles, 2 pedestrians, 1 two-wheels) với cao độ trung bình mean_z = 1.034 m.                  |
| C    | 1.73  | 0.32      | 6      | 1.091  | run-C/boxes-demo.json, run-C/side-demo.png, run-C/summary.csv | Khi tăng kích thước pillar XY lên 0.32 m làm giảm độ phân giải, số hộp giảm xuống còn 6 và tất cả đều bị phân loại thành pedestrian. |

- A/B — chỉ đổi delta: A có 1 hộp; B có 13 hộp. Ảnh/file/vùng side-demo.png trong run-A và run-B khác ở cao độ phân bố đám mây điểm so với mặt đất danh định của mạng nơ-ron, dẫn đến việc kích hoạt phát hiện hộp tăng từ 1 lên 13. Đây là chạy lại model trên input khác, không chỉ dịch hộp cũ; điều em còn chưa chắc là độ nhạy của phép ước lượng ground từ dữ liệu thô đối với các frame có độ dốc lớn.
- B/C — chỉ đổi pillar: B có 13 hộp; C có 6 hộp. Ảnh/file/vùng side-demo.png và boxes-demo.json khác ở việc mất toàn bộ bounding box của xe lớn và phương tiện 2 bánh, chỉ còn 6 detection nhỏ. Số lượng/lớp/vị trí thay đổi như sau: số lượng giảm từ 13 xuống 6 hộp, lớp bị gom sai thành pedestrian và kích thước bounding box bị co cụm lại. Có đủ bằng chứng để kết luận tốt hơn không? Không, pillar lớn làm mất đặc trưng hình học chi tiết nên kết quả ở lượt C kém hơn baseline B dù thời gian gom điểm có thể nhanh hơn.
- Giới hạn ROI và góc Side ảnh hưởng cách đọc miss/yaw thế nào? Góc nhìn Side view chỉ cho phép kiểm tra cao độ z và chiều dài/cao của đối tượng, không thể hiện được góc xoay hướng yaw trên mặt phẳng ngang cũng như dễ bỏ sót các vật thể bị che khuất theo chiều sâu nếu không đối chiếu phối hợp cùng Top view (BEV) và ảnh camera.
- JSON nào còn chưa đủ cơ sở để import? Cả 3 file JSON của A, B, C đều không được import vào CVAT Robotaxi vì đây là kết quả của model pretrained KITTI trên frame thử nghiệm, khác biệt về nhãn, hệ tọa độ và camera calibration so với dữ liệu thực tế Robotaxi. Cần kiểm tra pipeline và nạp pre-label từ portal của hệ thống.

## Ca QC có kiểm soát — không import CVAT

| Ca             | Số hộp lệch z / tổng hộp | Lượng lệch                 | Class/x/y/yaw có đổi? | Dừng batch, kiểm từng hộp hay chưa rõ? | Bằng chứng                                                                                                    |
| -------------- | ------------------------ | -------------------------- | --------------------- | -------------------------------------- | ------------------------------------------------------------------------------------------------------------- |
| case-correct   | 0 / 13                   | 0 m                        | Không đổi             | Không lỗi batch, giữ nguyên            | Dữ liệu trùng khớp hoàn toàn với baseline B (13 hộp chuẩn).                                                   |
| case-batch-z   | 13 / 13                  | Lệch đồng loạt theo trục z | Không đổi             | Dừng batch, kiểm pipeline/transform z  | Toàn bộ 13/13 hộp bị nhấc bổng/hạ thấp cùng một khoảng z; lỗi do sai lệch ma trận chuyển đổi tọa độ hệ thống. |
| case-one-box-z | 1 / 13                   | Lệch z tại một đối tượng   | Không đổi             | Kiểm từng hộp/đối tượng lẻ             | 12 hộp đúng vị trí mặt đất, chỉ có duy nhất 1 hộp bị lệch cao độ; cần mở 3D view để chỉnh riêng hộp đó.       |

Ghi rõ helper tạo biến đổi có chủ đích từ prediction, không phải kết quả inference riêng hoặc nhãn đúng.

## Nhận xét cá nhân

- Võ Thành Đức (2A202602167):
  - Vai trò đã làm: Trực tiếp thao tác lệnh runner trên Windows PowerShell, kiểm soát container Docker native amd64, trích xuất dữ liệu summary.csv và đối chiếu các ảnh side-demo.png cùng boxes-demo.json của cả 3 cấu hình A/B/C.
  - Quan sát A/B/C có dẫn chứng: Lượt A (delta=0) chỉ tìm được 1 xe do không bù trừ độ cao cảm biến khiến điểm rơi ra ngoài vùng nhận diện của model pretrained KITTI. Lượt B (delta=1.73) khôi phục đúng mặt đất nên nhận diện đầy đủ 13 đối tượng (10 vehicles, 2 pedestrians, 1 two-wheels). Lượt C (pillar=0.32) làm suy giảm độ phân giải không gian, khiến 7 đối tượng bị biến mất và 6 đối tượng còn lại bị gán nhãn sai thành pedestrian.
  - Diễn giải phép z thuận/ngược: Phép dịch z thuận can thiệp vào đám mây điểm đầu vào trước khi qua mạng PointPillars, làm thay đổi hoàn toàn quá trình tạo voxel và số lượng hộp phát hiện; khác với việc sau khi có hộp rồi mới cộng trừ cao độ z (phép biến đổi ngược/hậu xử lý).
  - Quyết định lỗi batch: Khi gặp case-batch-z, quyết định dứt khoát là dừng toàn bộ batch để kiểm tra lại pipeline tiền xử lý và ma trận extrinsic/transform z, tuyệt đối không chỉnh sửa thủ công từng hộp trên CVAT.
  - Điều chưa chắc: Ảnh hưởng của việc loại bỏ intensity/reflectance thực tế đối với khả năng nhận diện các vật thể có bề mặt hấp thụ sóng LiDAR ở khoảng cách xa.

## LC ghi nhận riêng

- Quyền dùng PCD/image và đúng ca:
- Có chạy thật / chỉ phân tích; còn cần lượt thực hành bổ sung:
- Output đủ, giữ bản gốc, không đưa ca lỗi vào CVAT:
- Nhận xét từng thành viên và quyết định dừng pipeline:
- Đồng ý chuyển sang chỉnh/QC / cần bổ sung; lý do:
