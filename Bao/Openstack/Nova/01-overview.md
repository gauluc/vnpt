# Tổng quan về OpenStack Nova

## 1. Khái niệm, các thành phần, kiến trúc và mối quan hệ

### 1.1 Khái niệm
**OpenStack Nova** (Compute Service) là trái tim của hệ thống OpenStack. Nó chịu trách nhiệm lớn nhất: quản lý vòng đời của các máy ảo (VM) – từ lúc cấp phát (provisioning), lập lịch (scheduling), khởi động, tạm dừng, đến khi xóa bỏ máy ảo. Nova đóng vai trò như một bộ điều phối (orchestrator), ra lệnh cho các hypervisor (như KVM, QEMU, VMware) thực thi việc tạo máy ảo.

### 1.2 Các thành phần và kiến trúc
Kiến trúc Nova có tính phân tán, bao gồm các service (daemon) độc lập giao tiếp với nhau thông qua hàng đợi bản tin (Message Queue - thường là RabbitMQ) và cơ sở dữ liệu dùng chung.
- **nova-api:** Điểm vào (entry point) của mọi yêu cầu. Chấp nhận và phản hồi các lệnh gọi API từ phía người dùng (qua REST, CLI).
- **nova-compute:** "Công nhân" thực sự. Nó chạy trên các Compute Node (các server vật lý chứa máy ảo). Nó giao tiếp trực tiếp với Hypervisor (thông qua thư viện libvirt) để tạo, xóa, bật, tắt máy ảo.
- **nova-scheduler:** Quyết định xem một máy ảo mới sẽ được tạo trên Compute Node nào dựa trên tài nguyên hiện có và các bộ lọc (filters).
- **nova-conductor:** Đóng vai trò cầu nối giữa `nova-compute` và cơ sở dữ liệu. Giúp tăng bảo mật vì `nova-compute` (nơi có khả năng bị tấn công cao hơn) không thể trực tiếp chọc vào DB.
- **nova-novncproxy / nova-consoleauth:** Cấp quyền truy cập trực tiếp vào màn hình máy ảo thông qua giao diện web (VNC).
- **Placement Service (Keystone/Nova Placement):** (Được tách ra từ Nova API) Theo dõi, kiểm kê tài nguyên (Resource inventory) hiện có và độ khả dụng của chúng (CPU, RAM, Disk).

## 2. Tìm hiểu file cấu hình Nova

File cấu hình chính của Nova thường nằm ở `/etc/nova/nova.conf`. Nó rất lớn và chia thành nhiều section:
- `[DEFAULT]`: Cấu hình Message Queue (RabbitMQ), định tuyến API.
- `[api_database]` và `[database]`: Kết nối tới 2 database của nova (nova_api DB và nova cell DB).
- `[glance]`, `[neutron]`, `[cinder]`, `[keystone_authtoken]`: Cấu hình thông tin xác thực để Nova gọi sang các dịch vụ khác tương ứng (Image, Network, Volume, Identity).
- `[vnc]`: Cấu hình VNC (IP lắng nghe, URL VNC server proxy).
- `[libvirt]`: (Nằm trên compute node) Chỉ định loại ảo hóa đang dùng (ví dụ: `virt_type = kvm` hoặc `qemu`).

## 3. Quá trình khởi tạo máy ảo

Khi người dùng chạy lệnh tạo máy ảo, luồng đi như sau:
1. **nova-api:** Nhận yêu cầu tạo máy ảo. Kiểm tra xác thực (Keystone). Ghi bản ghi vào Database ở trạng thái "building". Đẩy bản tin lên Message Queue.
2. **nova-conductor:** Bắt bản tin từ Queue. Thấy cần tạo máy ảo mới, nó gọi sang **nova-scheduler** để xin vị trí.
3. **nova-scheduler:** Lấy thông tin tài nguyên từ DB và **Placement API**. Chạy qua các bộ lọc (Filters) và điểm số (Weights) để chọn ra một Compute Node phù hợp nhất. Trả kết quả về cho conductor.
4. **nova-conductor:** Đẩy một bản tin xuống Queue dành riêng cho Compute Node đã được chọn.
5. **nova-compute (trên Node đích):** Nhận lệnh. Nó bắt đầu tương tác với các service khác:
   - Gọi **Neutron** để lấy IP và Port mạng.
   - Gọi **Cinder** để gắn ổ đĩa (nếu dùng block storage).
   - Gọi **Glance** để tải Image hệ điều hành về ổ cứng cục bộ.
6. **nova-compute** dịch các thông số này thành cấu hình XML của libvirt và ra lệnh cho Hypervisor khởi động máy ảo. Trạng thái cập nhật thành `Active`.

## 4. Tìm hiểu Cloud-init

### 4.1 Khái niệm
**Cloud-init** là một gói phần mềm (tiêu chuẩn công nghiệp) được cài sẵn bên trong các bản phân phối Linux OS images. Nó chạy ngay khi máy ảo vừa boot lên lần đầu tiên (Early initialization) để thực hiện cấu hình tự động (như gán IP, đổi hostname, tạo user, nạp SSH key,...).

### 4.2 Các module và ví dụ
Nova truyền thông tin cho Cloud-init thông qua ổ đĩa CD ảo cấu hình (ConfigDrive) hoặc qua dịch vụ Metadata.
- **Cấu hình mạng:** Tự gán IP theo thông tin từ DHCP của Neutron.
- **SSH Key:** Tự động nạp public key của người dùng vào `~/.ssh/authorized_keys` của user mặc định (như `ubuntu`, `centos`).
- **User-data script:** Người dùng có thể truyền script bash tùy chỉnh vào lúc tạo VM.
  *Ví dụ User-data (bash):*
  ```bash
  #!/bin/bash
  apt update -y && apt install nginx -y
  ```
  *(Khi VM boot lên, Nginx sẽ tự được cài).*

## 5. Các lệnh thường dùng với Nova

*Lưu ý: Đã source openrc.*
- Xem danh sách máy ảo: `openstack server list`
- Xem chi tiết 1 máy ảo: `openstack server show <vm_id>`
- Xem danh sách các compute node: `openstack compute service list`
- Khởi động/Tắt/Reboot máy ảo: `openstack server start/stop/reboot <vm_id>`
- Xem log trên màn hình console máy ảo: `openstack console log show <vm_id>`
- Lấy đường dẫn VNC: `openstack console url show <vm_id>`

## 6. File log của Nova

Các log của Nova được chia nhỏ theo từng dịch vụ, thường nằm trong `/var/log/nova/`.
- `nova-api.log`: Lỗi khi giao tiếp REST API (như lỗi 400, 500 khi gõ lệnh openstack).
- `nova-compute.log`: Rất quan trọng! Dùng để debug khi tạo VM bị lỗi (ERROR trạng thái). Nó ghi lại các lỗi về libvirt, không cấp được mạng, không tải được ảnh.
- `nova-scheduler.log`: Lỗi khi trạng thái VM là `NoValidHost` (nghĩa là scheduler không tìm thấy compute node nào thỏa mãn yêu cầu).
- `nova-conductor.log`: Lỗi kết nối DB.

## 7. Luồng khởi tạo và quản lí VNC trong Nova

VNC (Virtual Network Computing) cho phép người dùng nhìn thấy và tương tác với màn hình máy ảo thông qua trình duyệt web.
**Luồng hoạt động:**
1. Trình duyệt gửi yêu cầu kết nối websocket tới `nova-novncproxy` (thường nằm ở node Controller).
2. `nova-novncproxy` yêu cầu `nova-consoleauth` xác thực quyền truy cập của user.
3. Nếu hợp lệ, proxy sẽ kết nối trực tiếp đến địa chỉ IP nội bộ của Compute Node (nơi máy ảo đang chạy) tại port VNC tương ứng do QEMU/KVM cấp phát (VD: 5900).
4. Proxy chuyển đổi tín hiệu từ VNC thuần sang Websocket để hiển thị lên trình duyệt cho người dùng. Người dùng hoàn toàn không cần tiếp xúc với IP thật của Compute node.

## 8. Tại sao lại cần Placement và Conductor?

- **nova-conductor:** Trong mô hình cũ, `nova-compute` chọc trực tiếp vào Database. Do `nova-compute` nằm trên các server chứa máy ảo (nguy cơ bị hacker tấn công cao), nếu node này bị chiếm quyền, hacker có thể lấy thông tin DB. `nova-conductor` sinh ra làm proxy (cầu nối). `nova-compute` chỉ giao tiếp với `conductor` qua message queue, và chỉ `conductor` mới có quyền đọc/ghi DB. Điều này tăng cường bảo mật.
- **nova-placement:** Ban đầu, scheduler lấy thông tin tài nguyên (CPU/RAM còn lại) trực tiếp từ Nova DB. Khi scale lên hàng ngàn node, việc query DB trở nên cực kỳ chậm. `Placement API` được tách ra thành một REST service siêu nhẹ chuyên chỉ để lưu trữ và truy vấn "Kho tài nguyên" (Inventory). Giúp quá trình Schedule nhanh hơn gấp nhiều lần.

## 9. Host Aggregate vs Availability Zone

- **Host Aggregate (Cụm máy chủ):** Là một khái niệm trừu tượng dùng riêng cho Admin. Dùng để gom nhóm các Compute Node có chung đặc điểm phần cứng (VD: Nhóm máy chủ có GPU, Nhóm dùng ổ SSD). Người dùng thường không biết tới khái niệm này.
- **Availability Zone (Vùng khả dụng - AZ):** Là khái niệm dành cho End-user. AZ được gắn lên trên các Host Aggregate. Người dùng khi tạo máy ảo có thể chọn AZ (VD: `Zone1`, `Zone2`) để đảm bảo các máy ảo của mình chạy ở các trung tâm dữ liệu/tòa nhà khác nhau nhằm dự phòng thảm họa (HA).

## 10. Cơ chế Scheduler và Filter trong Nova

Khi có yêu cầu tạo máy ảo, **nova-scheduler** sẽ tìm Node phù hợp thông qua 2 bước:
1. **Filtering (Lọc):** Loại bỏ các node không thỏa mãn. Có nhiều bộ lọc (như `RamFilter`: loại các node hết RAM, `ComputeFilter`: loại các node đang chết, `ServerGroupAntiAffinityFilter`: không đặt 2 VM cụ thể lên cùng 1 node). Node nào vượt qua hết các filter sẽ lọt vào danh sách ứng viên.
2. **Weighing (Chấm điểm):** Trong các ứng viên, scheduler sẽ chấm điểm dựa trên trọng số (Weight). VD: Node nào còn nhiều RAM nhất sẽ được điểm cao nhất.
Cuối cùng, Node có điểm cao nhất sẽ được chọn để tạo máy ảo. Nếu sau bước Filter không còn node nào, Nova sẽ báo lỗi `NoValidHost`.

## 11. Các khái niệm liên quan đến di chuyển máy ảo trong Ops

Đây là các thao tác vận hành (Ops) đối với VM:
- **Cold Migrate:** Tắt máy ảo -> Di chuyển dữ liệu sang Compute Node khác -> Bật lên. Gây gián đoạn dịch vụ.
- **Live Migrate:** Chuyển máy ảo sang Compute Node khác trong khi máy ảo **vẫn đang chạy**. Gần như không gây gián đoạn mạng. (Thường yêu cầu Shared Storage hoặc block-live-migration).
- **Resize:** Giống như Cold Migrate nhưng mục đích chính là thay đổi cấu hình (Flavor) của máy ảo (tăng/giảm CPU, RAM). Máy ảo sẽ khởi động lại với thông số phần cứng ảo mới.
- **Evacuate:** Được sử dụng khi một Compute Node **đã chết hẳn** (bị cháy hỏng). Nova sẽ tái tạo lại máy ảo đó sang một Compute Node khác còn sống. Yêu cầu ổ đĩa gốc của máy ảo phải nằm ở Shared Storage hoặc Cinder.
- **Rescue:** Chế độ "cứu hộ". Máy ảo sẽ được boot bằng một file ảnh tạm thời (chứa hệ điều hành cứu hộ), còn ổ cứng gốc bị lỗi sẽ được gắn như là một ổ đĩa thứ hai. Admin ssh vào để sửa chữa các file cấu hình bị hỏng, sau khi xong thì thoát khỏi rescue để boot lại ổ gốc.
