# Tổng quan về OpenStack Glance

## 1. Khái niệm và Kiến trúc

### 1.1 Khái niệm
**OpenStack Glance** (Image Service) là dịch vụ trung tâm chịu trách nhiệm khám phá (discovering), đăng ký (registering), và lấy lại (retrieving) các file hệ điều hành ảo (virtual machine images). Nó cung cấp một API RESTful cho phép truy vấn siêu dữ liệu (metadata) của image cũng như lấy chính các image đó. 
Nói một cách đơn giản, Glance đóng vai trò là kho lưu trữ các bản cài đặt hệ điều hành mẫu để dịch vụ Compute (Nova) có thể tải về và sử dụng để tạo máy ảo mới.

### 1.2 Kiến trúc
Glance có kiến trúc client-server và bao gồm các thành phần chính sau:
- **glance-api:** Đây là thành phần tiếp nhận các API request từ người dùng (như tìm kiếm, tải lên, lấy thông tin image). Nó điều phối luồng làm việc đến các thành phần khác.
- **glance-registry:** Chịu trách nhiệm lưu trữ, xử lý và lấy lại siêu dữ liệu (metadata) của image (ví dụ: kích thước, định dạng, ID). Thông tin này thường được lưu trong cơ sở dữ liệu SQL (như MariaDB/MySQL).
- **Database:** Lưu siêu dữ liệu (metadata) của image.
- **Storage Repository (Backend Store):** Glance không có bộ lưu trữ riêng mà dựa vào các backend khác nhau để lưu nội dung file image thực sự. Các backend phổ biến bao gồm: file system cục bộ, Object Storage (Swift), Block Storage (Cinder), Ceph RBD, S3, hoặc HTTP.

## 2. Một số định dạng Image được hỗ trợ

Khi tải một image lên Glance, bạn cần xác định **Disk Format** (định dạng ổ cứng) và **Container Format** (định dạng đóng gói).
- **Disk Formats phổ biến:**
  - `qcow2`: (QEMU copy-on-write) Định dạng phổ biến nhất, hỗ trợ cấp phát động (thin provisioning) và snapshot, thường dùng với KVM.
  - `raw`: Định dạng thô, không cấu trúc. Tốc độ đọc/ghi cao nhất nhưng tốn nhiều dung lượng lưu trữ do không hỗ trợ cấp phát động.
  - `vhd` / `vhdx`: Định dạng ảo hóa của Microsoft Hyper-V.
  - `vmdk`: Định dạng ảo hóa của VMware.
  - `iso`: Định dạng file ảnh của đĩa CD/DVD, thường dùng để cài đặt OS ban đầu.
- **Container Formats:** Thường là `bare` (nếu không có đóng gói nào) hoặc `ovf` (Open Virtualization Format).

## 3. Các trạng thái và luồng hoạt động của Image trong Glance

### 3.1 Các trạng thái của Image
Trong quá trình tồn tại, một image có thể trải qua các trạng thái (status) sau:
- `queued`: Image mới được tạo ra trên Database nhưng chưa có dữ liệu thực sự tải lên.
- `saving`: Đang trong quá trình tải dữ liệu file image vào backend lưu trữ.
- `active`: Image đã tải lên thành công hoàn toàn và sẵn sàng để sử dụng (ví dụ: cấp phát cho Nova).
- `killed`: Có lỗi xảy ra trong quá trình upload (ví dụ: mạng đứt, lỗi backend), image không thể sử dụng.
- `deleted`: Image đã bị xóa (Glance vẫn lưu thông tin ở database nhưng đánh dấu là deleted).
- `pending_delete`: Tương tự như deleted, nhưng chờ quá trình dọn dẹp thực sự ở phía backend (nếu có tính năng scrub rác).

### 3.2 Luồng hoạt động cơ bản (Upload Image)
1. Người dùng chạy lệnh `openstack image create` tải lên một image.
2. Request đi đến `glance-api`.
3. `glance-api` làm việc với `glance-registry` (hoặc trực tiếp ghi DB ở các bản OpenStack mới) để tạo một bản ghi cho image ở trạng thái `queued`.
4. Dữ liệu file image bắt đầu được truyền trực tiếp đến backend storage. Trạng thái chuyển sang `saving`.
5. Khi upload hoàn tất, `glance-api` cập nhật lại trạng thái thành `active` trong DB.

## 4. File cấu hình của Glance

Các file cấu hình của Glance thường nằm tại thư mục `/etc/glance/`. Các file quan trọng bao gồm:
- **`glance-api.conf`:** Cấu hình chính cho API server. 
  - `[database]`: Định nghĩa chuỗi kết nối SQL Database.
  - `[keystone_authtoken]`: Cấu hình xác thực với Keystone.
  - `[glance_store]`: Cấu hình backend lưu trữ (ví dụ `stores = file,http`, `default_store = file`, `filesystem_store_datadir = /var/lib/glance/images`).
- **`glance-registry.conf`:** Cấu hình cho registry server (trong các bản OpenStack cũ, từ bản Wallaby trở đi thành phần này bị loại bỏ dần vì `glance-api` sẽ ghi trực tiếp vào DB).
- **`glance-image-sync.conf`:** (Nếu có) Dùng để đồng bộ image giữa nhiều node.

## 5. File log của Glance

Các file log là nơi phân tích, debug lỗi nếu không tải được image hoặc Nova không tải được image về.
- Vị trí phổ biến: `/var/log/glance/`
- File quan trọng:
  - `/var/log/glance/glance-api.log`: Ghi lại mọi giao dịch, lỗi khi người dùng hoặc service khác gọi API của Glance.
  - `/var/log/glance/glance-registry.log`: Log quá trình tương tác với Database (nếu sử dụng).

*Mẹo: Để xem log chi tiết nhất, hãy bật `debug = true` trong file `glance-api.conf`.*

## 6. Một số command thường dùng

*Lưu ý: Source file môi trường trước khi chạy (ví dụ `source openrc`).*

- Liệt kê danh sách image đang có:
  `openstack image list`
- Xem thông tin chi tiết của một image:
  `openstack image show <image_name_or_id>`
- Tạo mới và tải lên một image:
  ```bash
  openstack image create "Ubuntu 22.04" \
    --file ./ubuntu-22.04-server-cloudimg-amd64.img \
    --disk-format qcow2 \
    --container-format bare \
    --public
  ```
  *(Tham số `--public` cho phép tất cả các Project khác đều có thể nhìn thấy image này).*
- Xóa một image:
  `openstack image delete <image_name_or_id>`
- Sửa đổi các thuộc tính của một image (ví dụ đổi tên):
  `openstack image set --name "New Ubuntu Name" <image_id>`

## 7. Metadata của image

- **Khái niệm:** Metadata là các thuộc tính bổ sung gắn liền với một image để mô tả rõ hơn về nó, giúp bộ lập lịch của Nova (Nova Scheduler) hoặc người dùng có thêm thông tin.
- **Các metadata cơ bản:** Thuộc tính sẵn có như `id`, `name`, `disk_format`, `container_format`, `size`, `status`, `min_ram` (RAM tối thiểu để chạy image), `min_disk` (ổ cứng tối thiểu để chạy image).
- **Thuộc tính tuỳ chỉnh (Custom Properties):** Có thể gán thêm các metadata tự định nghĩa dưới dạng key-value. 
  - Ví dụ: Thêm thuộc tính `hw_scsi_model=virtio-scsi` để báo cho Nova dùng chuẩn kết nối ổ đĩa nào.
  - Thêm thuộc tính `hw_qemu_guest_agent=yes` để báo hiệu bên trong image có cài sẵn agent hỗ trợ.
- **Cập nhật metadata:** 
  `openstack image set --property key=value <image_id>`
