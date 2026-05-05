# Tổng quan về OpenStack Cinder

## 1. Tổng quan, các thành phần và backend thông dụng

### 1.1 Tổng quan
**Cinder** (Block Storage Service) là dịch vụ cung cấp lưu trữ dạng khối (block storage) cho các máy ảo (VM) trong OpenStack. Khác với ổ cứng ảo dùng một lần (Ephemeral storage - bị mất dữ liệu khi máy ảo bị xóa), dữ liệu trên Cinder volume mang tính bền vững (Persistent). Volume tồn tại độc lập với vòng đời của máy ảo; bạn có thể gắn (attach) và tháo (detach) nó vào bất kỳ máy ảo nào giống như một chiếc ổ cứng di động (USB).

### 1.2 Các thành phần trong Cinder
Kiến trúc Cinder bao gồm các daemon độc lập giao tiếp với nhau qua Message Queue (RabbitMQ):
- **cinder-api:** Nhận và điều hướng mọi request API (tạo, xóa, attach, snapshot volume).
- **cinder-scheduler:** Tương tự `nova-scheduler`, có nhiệm vụ chọn Storage Node (backend) tối ưu nhất để đặt volume mới dựa trên sức chứa (capacity) và các bộ lọc (filters).
- **cinder-volume:** Dịch vụ "công nhân" thực thi việc tạo, xóa volume. Nó tương tác trực tiếp với các thiết bị hoặc hệ thống lưu trữ vật lý thông qua các trình điều khiển (drivers).
- **cinder-backup:** Dịch vụ dùng để sao lưu volume ra một hệ thống lưu trữ khác (như Swift, Ceph, NFS) để bảo vệ dữ liệu.

### 1.3 Một số backend thông dụng
Cinder hỗ trợ cơ chế plug-in nên kết nối được với nhiều loại storage:
- **LVM (Logical Volume Manager):** Quản lý ổ cứng cục bộ trực tiếp trên Storage Node. Thường dùng cho Lab vì tính đơn giản.
- **NFS (Network File System):** Lưu trữ chia sẻ file qua mạng.
- **Ceph (RBD):** Lựa chọn số 1 cho Production. Cung cấp lưu trữ phân tán, tốc độ cao, High Availability (HA) tuyệt vời.
- Storage chuyên dụng (SAN/NAS): IBM, Dell EMC, NetApp, HP.

## 2. Luồng hoạt động (Cinder Workflow)

**Ví dụ luồng tạo và gắn (Attach) Volume:**
1. **User** yêu cầu tạo volume -> `cinder-api` nhận, đẩy bản tin lên Queue.
2. `cinder-scheduler` nhận bản tin, đánh giá xem node backend nào trống và gửi lệnh.
3. `cinder-volume` tại node được chọn (VD: dùng LVM) sẽ cắt một phần Logical Volume (LV) bằng đúng dung lượng yêu cầu. Trạng thái volume chuyển thành `available`.
4. **User** yêu cầu attach volume vào VM. **Nova** gọi sang **Cinder API**.
5. `cinder-volume` sẽ cấu hình Export (VD: cấu hình iSCSI Target) và báo thông tin (IP, Port, IQN) lại cho Nova.
6. `nova-compute` (nơi máy ảo đang chạy) đóng vai trò iSCSI Initiator, kết nối mạng tới Storage Node. Sau khi kết nối, nó ánh xạ ổ cứng đó vào bên trong máy ảo (dưới dạng thiết bị khối, VD: `/dev/vdb`).

## 3. File cấu hình Cinder

File chính thường nằm ở `/etc/cinder/cinder.conf`. Các block quan trọng:
- `[DEFAULT]`: Kết nối Database, RabbitMQ, Keystone. Quan trọng nhất là chỉ thị `enabled_backends = lvm, nfs...` để báo cho Cinder biết đang dùng các backend nào.
- Các block do tự đặt tên (Ví dụ `[lvm]`, `[my_ceph]`): Chứa cấu hình kết nối cụ thể cho từng backend đó (driver name, iSCSI helper, pool name...).

## 4. Các câu lệnh thường dùng

*Lưu ý: Môi trường đã source `openrc`.*
- Tạo volume: `openstack volume create --size <size_GB> <volume_name>`
- Xem danh sách: `openstack volume list`
- Gắn vào máy ảo: `openstack server add volume <vm_id> <volume_id>`
- Gỡ khỏi máy ảo: `openstack server remove volume <vm_id> <volume_id>`
- Liệt kê các service Cinder để kiểm tra trạng thái hoạt động: `openstack volume service list`

## 5. Cinder Scheduler vs Cinder Backup

- **cinder-scheduler:** Có nhiệm vụ điều phối vị trí. Nó "chỉ tay" quyết định volume sẽ được tạo ở Storage Node nào để cân bằng tải và đảm bảo sức chứa, không làm việc với dữ liệu.
- **cinder-backup:** Làm việc trực tiếp với dữ liệu. Nó đọc dữ liệu từ một Volume hiện có (thường qua snapshot) rồi đóng gói và copy đẩy sang một hệ thống dự phòng (Backup repository) để lưu trữ an toàn dài hạn (khác với Snapshot thường được lưu trên chính backend gốc).

## 6. Thực hành: Cài đặt Cinder với backend là LVM (Tóm tắt)

1. Gắn một ổ cứng trống vào máy chủ Storage (VD: `/dev/sdb`).
2. Khởi tạo LVM Volume Group:
   ```bash
   pvcreate /dev/sdb
   vgcreate cinder-volumes /dev/sdb
   ```
3. Khai báo trong `cinder.conf`:
   ```ini
   [DEFAULT]
   enabled_backends = lvm
   
   [lvm]
   volume_driver = cinder.volume.drivers.lvm.LVMVolumeDriver
   volume_group = cinder-volumes
   target_protocol = iscsi
   target_helper = tgtadm
   volume_backend_name = LVM_BACKEND
   ```
4. Khởi động lại dịch vụ `cinder-volume`.

## 7. Thực hành: Cấu hình Cinder Backup với backend NFS

1. Chuẩn bị một thư mục share NFS trên 1 server (VD: `10.0.0.100:/cinder_backup`).
2. Trên node cài cinder-backup, sửa `cinder.conf`:
   ```ini
   [DEFAULT]
   backup_driver = cinder.backup.drivers.nfs.NFSBackupDriver
   backup_share = 10.0.0.100:/cinder_backup
   ```
3. Restart dịch vụ: `systemctl restart cinder-backup`.
4. Tạo backup test: `openstack volume backup create <volume_id>`.

## 8. Thực hành: Cài đặt Multi-Backend Cinder (NFS và LVM)

Bạn có thể chạy song song nhiều backend. Sửa `cinder.conf`:
```ini
[DEFAULT]
enabled_backends = lvm_1, nfs_1

[lvm_1]
volume_driver = cinder.volume.drivers.lvm.LVMVolumeDriver
volume_group = cinder-volumes
volume_backend_name = LVM_TYPE

[nfs_1]
volume_driver = cinder.volume.drivers.nfs.NfsDriver
nfs_shares_config = /etc/cinder/nfs_shares.txt
nfs_mount_point_base = /var/lib/cinder/mnt
volume_backend_name = NFS_TYPE
```
*(Trong file `nfs_shares.txt` ghi địa chỉ IP và path của NFS server).* Restart `cinder-volume`.

## 9. Thực hành: Schedule Volume theo từng backend (Volume Type)

Khi có Multi-backend, làm sao hệ thống biết tạo volume vào NFS hay LVM? Giải pháp là cấu hình **Volume Type**:
1. Tạo 2 type: 
   `openstack volume type create lvm-disk`
   `openstack volume type create nfs-disk`
2. Liên kết Type với `volume_backend_name` đã cấu hình ở bước 8:
   `openstack volume type set --property volume_backend_name=LVM_TYPE lvm-disk`
   `openstack volume type set --property volume_backend_name=NFS_TYPE nfs-disk`
3. Khi tạo Volume, User chọn Type:
   `openstack volume create --type nfs-disk --size 10 my_nfs_volume`

## 10. File Log của Cinder

Vị trí: `/var/log/cinder/`
- `cinder-api.log`: Xem lỗi khi gõ lệnh (lỗi 400, auth 401).
- `cinder-scheduler.log`: Debug các lỗi cấp phát (VD: `No valid host was found` do backend đầy hoặc khai báo sai).
- `cinder-volume.log`: **Quan trọng nhất.** Chứa chi tiết quá trình liên kết với ổ đĩa vật lý, quá trình map iSCSI lỗi, tạo LVM VG lỗi,...
- `cinder-backup.log`: Log quá trình đẩy dữ liệu backup qua mạng.

## 11. Thực hành: Migrate và Evacuate với máy ảo boot từ Volume

Máy ảo có 2 cách khởi động: boot từ local disk (của Nova) hoặc boot từ Cinder Volume (OS nằm trên SAN/Ceph).
Boot từ Volume mang lại lợi ích vận hành (Ops) khổng lồ:
- **Live Migrate siêu tốc:** Vì toàn bộ hệ điều hành (OS) nằm trên Cinder (lưu trữ chia sẻ - shared storage), việc Live Migrate máy ảo sang Compute Node khác diễn ra cực kỳ nhanh. Hệ thống không cần copy hàng chục GB ổ cứng qua mạng. Node mới chỉ cần attach lại đúng Volume đó vào hypervisor và chạy.
- **Evacuate an toàn:** Khi một Compute Node phần cứng chết cháy. Bạn có thể evacuate (sơ tán) máy ảo sang node khác. Toàn bộ dữ liệu OS an toàn trên Cinder. Node mới khởi động máy ảo lên, cắm Volume OS vào và hệ thống phục hồi lập tức mà không mất mát dữ liệu local.
