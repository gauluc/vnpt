# Tổng quan về OpenStack Keystone

## 1. Tìm hiểu các khái niệm về Project, User, Domain, Role

- **User (Người dùng):** Đại diện cho một cá nhân, hệ thống hoặc dịch vụ sử dụng OpenStack. Một User có thông tin xác thực (thường là username/password) để đăng nhập và gọi API.
- **Project (Dự án) / Tenant:** Là đơn vị cơ bản để gom nhóm và cô lập tài nguyên (máy ảo, mạng, ổ đĩa ảo,...). Bất kỳ tài nguyên nào được tạo ra trong OpenStack đều thuộc về một Project. Một User có thể thuộc về một hoặc nhiều Project.
- **Domain (Miền):** Mức phân cấp cao nhất, được sử dụng để gom nhóm các Project, User và Group. Domain giúp chia tách quản lý cho các môi trường lớn (ví dụ: mỗi công ty/khách hàng/phòng ban là một Domain). Domain cung cấp một không gian tên (namespace) độc lập.
- **Role (Vai trò):** Định nghĩa các quyền hạn (permissions) hoặc cấp độ truy cập. Role thường được gán cho một User (hoặc Group) trên một phạm vi cụ thể (trên một Project hoặc một Domain). Ví dụ: `admin`, `member`, `reader`.

## 2. Vai trò của Keystone

Keystone là dịch vụ Định danh (Identity Service) cốt lõi của OpenStack. Nó đóng các vai trò chính:
- **Authentication (Xác thực):** Xác minh danh tính của người dùng hoặc các dịch vụ khác thông qua thông tin đăng nhập.
- **Authorization (Phân quyền):** Cung cấp thông tin về Role của người dùng, làm cơ sở để các dịch vụ khác (như Nova, Neutron) cho phép hoặc từ chối thực hiện một hành động.
- **Service Catalog (Danh mục dịch vụ):** Lưu trữ danh sách các dịch vụ đang chạy trong OpenStack và đường dẫn (Endpoint) để người dùng biết gửi API request tới đâu.
- **Identity Management:** Cung cấp cơ chế để quản lý vòng đời của User, Group, Project, Role, Domain.

## 3. Các thành phần và kiến trúc của Keystone

Kiến trúc Keystone bao gồm các thành phần (backend) chính sau:
- **Identity:** Cung cấp thông tin xác thực về User và Group (dữ liệu có thể lưu ở SQL database hoặc backend LDAP/Active Directory).
- **Resource:** Cung cấp dữ liệu về Project và Domain.
- **Assignment:** Cung cấp dữ liệu về Role-grant (ai có Role gì trên Project/Domain nào).
- **Token:** Quản lý vòng đời (tạo, xác thực) của các token sau khi đăng nhập thành công.
- **Catalog:** Cung cấp danh sách Endpoint của các dịch vụ OpenStack.
- **Policy:** Cung cấp quy tắc phân quyền (rule-based authorization engine).

## 4. Luồng làm việc cơ bản của Keystone

1. **Yêu cầu xác thực:** Người dùng gửi thông tin đăng nhập (username, password, domain, project) đến Keystone thông qua API.
2. **Kiểm tra thông tin:** Keystone kiểm tra thông tin với backend (database hoặc LDAP).
3. **Cấp Token & Catalog:** Nếu xác thực thành công, Keystone sinh ra một **Token** và trả về cho người dùng kèm theo **Service Catalog** (chứa URL của các dịch vụ).
4. **Gửi Request:** Người dùng sử dụng Endpoint (VD: URL của dịch vụ Nova từ Catalog) và đính kèm Token ở HTTP Header (`X-Auth-Token`) để yêu cầu tạo máy ảo.
5. **Xác thực Token (Token Validation):** Nova nhận request, nhưng không tự tin tưởng Token này. Nova gửi Token ngược lại Keystone (thông qua middleware) để xác minh xem Token này có hợp lệ không.
6. **Thực thi:** Keystone trả lời Token hợp lệ, kèm theo danh sách Role. Nova kiểm tra Policy (quy tắc phân quyền) và thực hiện việc tạo máy ảo nếu người dùng có đủ quyền.

## 5. Một số command OpenStack CLI hay sử dụng

*Lưu ý: Cần source file môi trường (VD: `source openrc.sh`) trước khi chạy.*
- Xem danh sách user: `openstack user list`
- Xem danh sách project: `openstack project list`
- Xem danh sách domain: `openstack domain list`
- Xem danh sách role: `openstack role list`
- Xem danh sách các dịch vụ: `openstack service list`
- Xem danh sách các endpoint: `openstack endpoint list`
- Gán role cho user trên một project: 
  `openstack role add --project <project_name> --user <user_name> <role_name>`
- Kiểm tra danh sách gán role: 
  `openstack role assignment list --user <user_name> --project <project_name> --names`

## 6. Các option trong file cấu hình của Keystone

File cấu hình chính thường nằm tại `/etc/keystone/keystone.conf`. Một số section quan trọng:
- `[database]`: Cấu hình kết nối DB (VD: `connection = mysql+pymysql://keystone:PASS@controller/keystone`).
- `[token]`: 
  - `provider = fernet` (chọn loại token).
  - `expiration = 3600` (thời gian sống của token tính bằng giây).
- `[identity]`: 
  - `driver = sql` (hoặc `ldap` để đồng bộ user từ Active Directory/LDAP).
- `[fernet_tokens]`: 
  - `key_repository = /etc/keystone/fernet-keys/` (thư mục chứa các khóa bảo mật để mã hóa token).

## 7. File log của Keystone

Log dùng để debug các lỗi liên quan đến xác thực (HTTP 401/403).
- Vị trí phổ biến: `/var/log/keystone/keystone.log`.
- Nếu Keystone chạy dưới dạng module WSGI của Apache/Nginx thì log nằm ở: `/var/log/apache2/keystone.log` hoặc `/var/log/httpd/keystone.log`.
- Để xem log chi tiết nhất, đổi cấu hình `debug = true` trong mục `[DEFAULT]` của `keystone.conf` và restart dịch vụ.

## 8. Khái niệm Endpoint và Catalog

- **Service Catalog:** Là "danh bạ" chứa danh sách tất cả các dịch vụ (Nova, Neutron, Cinder...) đang chạy. Nó được trả về cùng với Token sau khi User đăng nhập.
- **Endpoint:** Là địa chỉ URL cụ thể qua mạng để gọi API của một Service. Mỗi Service thường có 3 loại Endpoint (interface):
  - **Public:** URL mở ra cho người dùng cuối bên ngoài mạng (End-users) truy cập.
  - **Internal:** URL nội bộ dùng cho các dịch vụ OpenStack tự giao tiếp với nhau (VD: Nova gọi Neutron).
  - **Admin:** URL có đặc quyền dành cho quản trị viên cấu hình/sửa chữa.

## 9. Các loại Token và đặc điểm

Qua các phiên bản OpenStack, có nhiều loại Token đã được phát triển:
1. **UUID Token:** (Cũ) Chuỗi ngẫu nhiên (32 ký tự). Bắt buộc phải lưu vào Database của Keystone. Nhược điểm là Database sẽ phình to rất nhanh và quá tải khi cần xóa token cũ hoặc xử lý lượng request lớn.
2. **PKI / PKIZ Token:** (Cũ) Token mang theo toàn bộ thông tin (Payload) và được ký (Sign) bằng chứng chỉ X509. Dịch vụ (như Nova) có thể tự xác minh token mà không cần gọi lại Keystone. Nhược điểm: Kích thước token quá lớn (nhiều KB), gây lỗi HTTP Header khi truyền tải.
3. **Fernet Token:** (Chuẩn hiện tại) Khắc phục được nhược điểm của cả UUID và PKI. Không lưu vào Database và có kích thước siêu nhỏ (~255 bytes).

## 10. Fernet token, tại sao lại dùng Fernet

**Fernet** là một định dạng token mã hóa đối xứng (symmetric encryption) cung cấp bởi thư viện cryptography của Python. Nó trở thành token mặc định vì các ưu điểm tuyệt đối:
- **Stateless (Không lưu trữ):** Keystone không cần ghi Fernet token vào Database. Thay vào đó, nó chứa thẳng dữ liệu (User ID, Project ID, Thời hạn hết hạn...) và được **mã hóa**. Điều này giúp giải phóng hoàn toàn gánh nặng cho Database.
- **Kích thước cực nhỏ:** Dưới 300 bytes, loại bỏ tình trạng tràn HTTP Header.
- **Bảo mật:** Dữ liệu mã hóa chỉ có thể được giải mã bởi máy chủ nắm giữ bí mật (Fernet Keys). Nếu có ai đó bắt gói tin lấy được token, họ cũng không đọc được nội dung bên trong, và không thể làm giả Token nếu không có Key.

## 11. Cách sử dụng Token để giải mã và mã hóa

Trong cơ chế Fernet:
- **Mã hóa (Tạo Token):** Khi cấp Token, Keystone dùng "Khóa chính" (Primary Fernet key) nằm trong thư mục `/etc/keystone/fernet-keys/` để **mã hóa** thông tin User ID, Project ID thành chuỗi Fernet và trả về.
- **Giải mã (Xác minh Token):** Khi dịch vụ khác (Nova) nhận được token từ User, nó gọi API đến Keystone. Keystone sử dụng Fernet key để **giải mã**. Nếu khóa khớp, dữ liệu giải mã ra thời gian chưa hết hạn, Keystone sẽ báo Token hợp lệ.
*(Lưu ý: Cần thiết lập tiến trình cronjob chạy lệnh `keystone-manage fernet_rotate` định kỳ để xoay vòng các khóa mã hóa, nâng cao tính bảo mật).*

## 12. File cấu hình policy dạng JSON/YAML

- Policy là cơ chế kiểm soát truy cập (RBAC - Role-Based Access Control). Nó quy định Role nào được gọi API nào.
- Trước đây policy được lưu ở dạng file `policy.json`, ở các phiên bản gần đây OpenStack đổi sang định dạng `policy.yaml`.
- Vị trí: Mỗi dịch vụ có file riêng, ví dụ `/etc/keystone/policy.yaml`, `/etc/nova/policy.yaml`.
- Ví dụ về quy tắc: 
  `"identity:get_user": "rule:admin_required"` -> Chỉ User có role admin mới được xem thông tin user.
- **Điểm đặc biệt:** Khi chỉnh sửa file policy, sự thay đổi sẽ có tác dụng ngay lập tức mà không cần khởi động lại dịch vụ OpenStack đó.

## 13. Sử dụng CURL để gọi tới API của Keystone

### Dùng CURL giúp ta hình dung rõ nhất cách gọi RESTful API trong môi trường thực tế.

**Bước 1: Lấy token (Đăng nhập)**
```bash
curl -i -X POST http://<keystone_ip>:5000/v3/auth/tokens \
  -H "Content-Type: application/json" \
  -d '{
    "auth": {
      "identity": {
        "methods": ["password"],
        "password": {
          "user": {
            "name": "admin",
            "domain": { "id": "default" },
            "password": "your_password_here"
          }
        }
      },
      "scope": {
        "project": {
          "name": "admin",
          "domain": { "id": "default" }
        }
      }
    }
  }'
```
*Kết quả sẽ trả về HTTP 201 Created. **Token** chính là giá trị nằm ở Header `X-Subject-Token` trong kết quả trả về.*

**Bước 2: Sử dụng Token để gọi API khác (VD: Liệt kê danh sách users)**
Lấy chuỗi token vừa nhận được, đính kèm vào Header `X-Auth-Token` để gọi API lấy danh sách người dùng.
```bash
curl -s -X GET http://<keystone_ip>:5000/v3/users \
  -H "X-Auth-Token: <điền_token_vào_đây>" \
  | python3 -m json.tool
```

### Dùng python SDK của OpenStack để tương tác với API của Keystone:
```python
import openstack

# Kết nối tới Keystone
conn = openstack.connect(
    auth_url="http://<keystone_ip>:5000/v3",
    username="admin",
    password="your_password_here",
    project_name="admin",
    user_domain_name="default",
    project_domain_name="default"
)

# Liệt kê danh sách users
users = conn.users()
for user in users:
    print(user)
```