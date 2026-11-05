# Tổng quan về OpenStack Swift

Dựa theo bố cục của các thành phần trong OpenStack, dưới đây là tài liệu tổng quan về dịch vụ Swift (Object Storage).

## 1. Khái niệm cơ bản

### 1.1 Swift là gì?
**OpenStack Swift** là dịch vụ lưu trữ đối tượng (Object Storage) có khả năng mở rộng cực cao và dự phòng lỗi tốt (Highly Available & Fault Tolerant). 
Thay vì lưu trữ dữ liệu dưới dạng cấu trúc thư mục phân cấp (như File System) hay các khối (như Block Storage - Cinder), Swift lưu trữ dữ liệu dưới dạng các **"Đối tượng" (Objects)** trên một mặt phẳng (flat namespace) đi kèm với các siêu dữ liệu (metadata) mô tả đối tượng đó.
Tương tự như Amazon S3, Swift đặc biệt phù hợp để lưu trữ dữ liệu phi cấu trúc với dung lượng khổng lồ: file ảnh, video, log, file backup, hoặc image máy ảo.

## 2. Cấu trúc lưu trữ dữ liệu trong Swift

Dữ liệu trong Swift được sắp xếp theo một hệ thống phân cấp logic gồm 3 cấp độ:
- **Account (Tài khoản):** Cấp cao nhất, không phải là thông tin đăng nhập của người dùng mà là một vùng lưu trữ lớn nhất. Nó thường tương ứng với một Project/Tenant trong OpenStack.
- **Container (Thùng chứa):** Tương tự như khái niệm "Bucket" trong S3 hoặc thư mục (Directory). Tuy nhiên, Container không thể chứa một Container khác (không có hệ thống cây thư mục lồng nhau). Container dùng để gom nhóm các Object lại.
- **Object (Đối tượng):** Dữ liệu thực sự được lưu trữ (ví dụ: `image.jpg`, `video.mp4`) cộng với phần Metadata của nó. 

## 3. Các thành phần và Kiến trúc của Swift

Kiến trúc Swift có tính phân tán cao, chạy trên các cụm máy chủ (cluster) không dùng chung (shared-nothing architecture), bao gồm:

- **Proxy Server:** Đóng vai trò là cửa ngõ (API Gateway). Nó nhận tất cả các REST API requests từ người dùng (Upload, Download, Delete). Xác thực qua Keystone, sau đó sử dụng **Ring** để tìm vị trí máy chủ lưu trữ thực tế rồi định tuyến yêu cầu đến máy chủ đó.
- **Account Server:** Chịu trách nhiệm quản lý thông tin các Account. Nó sử dụng cơ sở dữ liệu SQLite để lưu danh sách các Container nằm trong Account đó.
- **Container Server:** Quản lý thông tin về Container (cũng bằng cơ sở dữ liệu SQLite). Lưu trữ danh sách các Object có trong Container đó. (Nó không lưu bản thân file Object).
- **Object Server:** "Công nhân" thực thụ lưu trữ nội dung file Object trên các ổ đĩa vật lý nội bộ của máy chủ. Nó lưu trữ dữ liệu dưới dạng file nhị phân trên hệ thống file cục bộ (thường dùng XFS).
- **The Ring:** Một thuật toán/bảng ánh xạ cốt lõi của Swift. Bất cứ khi nào nhận được một Object, Proxy Server nhìn vào Ring để tính toán và biết chính xác Object đó sẽ được lưu ở những ổ cứng nào, máy chủ nào, tủ rack (zone) nào. Có 3 file Ring riêng biệt: `account.ring`, `container.ring`, và `object.ring`.

## 4. Luồng hoạt động cơ bản (Đọc và Ghi)

**Cơ chế sao chép (Replication):** Swift sử dụng cơ chế eventual consistency. Mặc định, mỗi Object khi tải lên sẽ được Swift **tạo ra 3 bản sao (replicas)** và rải đều ra 3 ổ cứng trên 3 máy chủ hoặc 3 Zone khác nhau để đảm bảo dự phòng nếu có ổ cứng hay máy chủ bị cháy hỏng.

- **Luồng ghi (Upload Object):**
  1. User gọi API gửi file tới `Proxy Server`.
  2. `Proxy Server` nhìn vào cấu hình `object.ring` để tìm ra 3 máy chủ lưu trữ (Object Server) phù hợp nhất.
  3. Proxy gửi dữ liệu song song (hoặc tuần tự tùy cấu hình) đến cả 3 Object Server.
  4. Nếu đa số (ví dụ 2/3) máy chủ báo lưu thành công, Proxy báo cho User là Upload thành công. Bản thứ 3 nếu lúc đó rớt mạng sẽ tự động được các tiến trình nền (Replicator) đồng bộ bù vào sau đó.
- **Luồng đọc (Download Object):**
  1. User gọi API xin file tới `Proxy Server`.
  2. Proxy lại kiểm tra `object.ring` để biết 3 máy chủ nào đang giữ file này.
  3. Proxy sẽ kết nối tới máy chủ gần nhất hoặc rảnh nhất để lấy dữ liệu. Nếu máy chủ thứ nhất lỗi, nó sẽ tự động thử lấy ở máy chủ thứ 2. Sau đó trả dữ liệu về cho người dùng.

## 5. File cấu hình của Swift

Các cấu hình chính của Swift nằm ở thư mục `/etc/swift/`.
- `swift.conf`: File cấu hình chung, chứa chuỗi bí mật (Hash path suffix/prefix) dùng để tính toán mã hóa vị trí ổ đĩa. File này phải giống nhau trên mọi node.
- `proxy-server.conf`: Cấu hình cho Proxy node (IP lắng nghe, pipeline xác thực keystone, v.v.).
- `account-server.conf`, `container-server.conf`, `object-server.conf`: Cấu hình cho các dịch vụ lưu trữ (ví dụ đường dẫn mount ổ đĩa `/srv/node/`).
- Các file Ring: `account.ring.gz`, `container.ring.gz`, `object.ring.gz`. (Các file nhị phân này được sinh ra bởi lệnh `swift-ring-builder` và phải được copy giống hệt nhau sang mọi node Storage).

## 6. File log của Swift

Swift không sử dụng các file log riêng trong `/var/log/swift/` như các dịch vụ khác của OpenStack, mà mặc định Swift ghi toàn bộ log vào hệ thống **Syslog** (ví dụ `/var/log/syslog` hoặc `/var/log/messages`).
- Để debug Swift, bạn thường phải dùng lệnh `grep swift /var/log/syslog`.
- Admin thường cấu hình lại Rsyslog để gom riêng log của swift vào một file `/var/log/swift/swift.log` cho dễ đọc.

## 7. Các lệnh thường dùng

Có thể tương tác với Swift bằng lệnh `openstack` hoặc công cụ dòng lệnh chuyên dụng `swift`.

*Bằng lệnh openstack:*
- Lấy thông tin tài khoản (Account): `openstack object store account show`
- Tạo một container: `openstack container create <tên_container>`
- Liệt kê các container: `openstack container list`
- Tải file lên (Upload Object): `openstack object create <tên_container> <tên_file_ở_local>`
- Liệt kê các file trong container: `openstack object list <tên_container>`
- Tải file về máy (Download Object): `openstack object save <tên_container> <tên_object>`
- Xóa Object: `openstack object delete <tên_container> <tên_object>`

*Công cụ quản lý Ring (Dành cho Admin):*
- Xem thông tin vòng Ring: `swift-ring-builder /etc/swift/object.builder`

## 8. So sánh Swift (Object Storage) và Cinder (Block Storage)

| Tiêu chí | Cinder (Block Storage) | Swift (Object Storage) |
|---|---|---|
| **Cơ chế lưu trữ** | Ổ đĩa ảo. Chia theo block nhỏ để hệ điều hành đọc ghi. | Lưu trữ dạng đối tượng phẳng. Chỉ hỗ trợ đọc toàn bộ hoặc ghi toàn bộ file, không sửa một phần của file. |
| **Cách truy cập** | Gắn (Attach) trực tiếp vào một máy ảo duy nhất thông qua mạng nội bộ iSCSI/RBD. Truy cập qua File System của Linux (như `ext4`, `xfs`). | Không thể gắn trực tiếp vào máy ảo. Truy cập thông qua mạng Internet/HTTP bằng các API (RESTful: GET, PUT). |
| **Sử dụng chính** | Dùng làm ổ cứng cài Hệ điều hành (OS), làm ổ chứa Database cần đọc/ghi tốc độ cao, độ trễ thấp. | Dùng để lưu file tĩnh (ảnh, video), kho lưu trữ file Backup, kho chứa Image của Glance, web hosting tĩnh. |
| **Dung lượng mở rộng**| Bị giới hạn bởi số lượng máy chủ SAN/Volume. Khó mở rộng ngang (Scale-out) quy mô siêu lớn. | Khả năng mở rộng gần như vô hạn (Petabytes). Cứ cắm thêm ổ cứng, cắm thêm máy chủ là dung lượng tăng. |
