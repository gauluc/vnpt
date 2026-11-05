# Cloud-init 

## 1. Khái niệm,cấu hình, các thành phần và ví dụ

### 1.1 Khái niệm
Cloud-init là một công cụ khởi tạo máy ảo tiêu chuẩn công nghiệp trong điện toán đám mây. Nó là một gói phần mềm được cài đặt sẵn bên trong các bản phân phối Linux và chạy ngay lập tức khi máy ảo vừa được khởi động lần đầu tiên.

Mục đích chính của Cloud-init là tự động hóa các tác vụ cấu hình cơ bản cho máy ảo, giúp người dùng không cần phải đăng nhập vào máy và thiết lập thủ công. Nó cho phép máy ảo tự cấu hình mạng, nạp SSH key, cập nhật hệ thống, cài đặt ứng dụng, thay đổi hostname và chạy các script tùy chỉnh.

Cloud-init hỗ trợ nhiều nền tảng ảo hóa khác nhau như OpenStack, AWS, Azure, Google Cloud, VMware và các nền tảng đám mây khác.

### 1.2 Cấu hình và các thành phần

Cloud-init có cấu trúc modular với nhiều thành phần nhỏ (modules) xử lý các tác vụ khác nhau. Các module này được kích hoạt và cấu hình thông qua các file cấu hình. Cloud-init hỗ trợ nhiều cách để nhận cấu hình, bao gồm:

- User-data (dữ liệu người dùng): User-data có thể ở dạng văn bản thuần (text/plain), script bash, YAML hoặc JSON. Nó thường được sử dụng để chạy các lệnh tùy chỉnh, cài đặt phần mềm, cấu hình mạng, thiết lập SSH key, v.v.

- Network-config (cấu hình mạng): File cấu hình mạng cho các interface mạng của máy ảo. Có thể sử dụng Netplan, NetworkManager, hoặc các định dạng khác tùy thuộc vào hệ điều hành.

- Hostname: Thiết lập tên máy chủ (hostname) của máy ảo.

- Metadata (dữ liệu siêu): Thông tin được cung cấp bởi hypervisor (như tenant ID, hostname, public key, v.v.)

Một số thành phần chính của Cloud-init:

- **init** (khởi tạo): Component đầu tiên chạy, đọc metadata và user-data, sau đó khởi tạo các handler và config drive.
- **SSH** (Secure Shell): Module để cấu hình SSH, bao gồm nạp public key, tạo mật khẩu (nếu được cấu hình) và thiết lập quyền truy cập.
- **Network-config** (cấu hình mạng): Quản lý cấu hình mạng cho các interface, có thể sử dụng nhiều backend khác nhau.
- **Set-hostname** (thiết lập hostname): Đặt tên cho máy ảo.
- **Growpart** (mở rộng phân vùng): Tự động mở rộng phân vùng (thường là root partition) để sử dụng toàn bộ dung lượng đĩa có sẵn.
- **Config drive**: Một ổ đĩa CD ảo (iso) hoặc ổ đĩa block chứa user-data, network-config và metadata, được chèn vào máy ảo khi khởi tạo.

### 1.3 Ví dụ minh họa

Ví dụ về cấu hình user-data dưới dạng script bash để cài đặt và cấu hình web server:

```bash
#!/bin/bash

# Cập nhật hệ thống
apt update -y

# Cài đặt Nginx web server
apt install -y nginx

# Tạo file HTML đơn giản
cat <<EOF > /var/www/html/index.html
<html>
<head><title>Cloud-init Test</title></head>
<body>
    <h1>Hello from Cloud-init!</h1>
    <p>This VM was automatically configured on boot.</p>
</body>
</html>
EOF

# Khởi động và kích hoạt Nginx
systemctl enable nginx
systemctl start nginx
```

Cấu hình user-data dưới dạng YAML (thường được sử dụng trong OpenStack):

```yaml
#cloud-config

# Thiết lập hostname
hostname: my-ubuntu-vm

# Cấu hình mạng với Netplan
network:
  version: 2
  ethernets:
    eth0:
      dhcp4: true
      dhcp6: true

# Cấu hình SSH public key
ssh_authorized_keys:
  - ssh-rsa AAAAB3NzaC1yc2EAAAADAQ... user@example.com

# Script để chạy sau khi boot
runcmd:
  - [ "apt", "update", "-y" ]
  - [ "apt", "install", "-y", "nginx" ]
  - [ "systemctl", "enable", "nginx" ]
  - [ "systemctl", "start", "nginx" ]
```

Cấu hình network-config sử dụng Netplan (Ubuntu):

```yaml
#cloud-config
network:
  version: 2
  ethernets:
    eth0:
      dhcp4: true
      dhcp6: false
    eth1:
      addresses: ["10.0.0.100/24"]
      gateway4: 10.0.0.1
      nameservers:
        addresses: [8.8.8.8, 8.8.4.4]
```

## 2. Các loại Cloud-init khác nhau trong OpenStack

Trong OpenStack, cloud-init được sử dụng để cung cấp thông tin cấu hình ban đầu cho máy ảo. Có ba loại cloud-init chính được sử dụng trong OpenStack:

### 2.1 User-data

User-data là dữ liệu do người dùng cung cấp khi tạo máy ảo, có thể ở dạng:

- Văn bản thuần (text/plain): Chứa các lệnh script bash hoặc các lệnh cấu hình đơn giản.
- YAML/JSON: Có cấu trúc và hỗ trợ các trường phức tạp hơn như cấu hình mạng, SSH key, v.v.
- ISO (ISO image): Chứa nhiều file cấu hình, cho phép cấu hình phức tạp hơn.

User-data thường được sử dụng để:

- Cài đặt và cấu hình ứng dụng
- Thiết lập thông tin đăng nhập (SSH key, mật khẩu)
- Thay đổi cấu hình hệ thống
- Chạy các script tùy chỉnh

Ví dụ về user-data YAML trong OpenStack:

```yaml
#cloud-config

# Thiết lập thông tin đăng nhập
ssh_authorized_keys:
  - ssh-rsa AAAAB3NzaC1yc2EAAAADAQ... user@example.com

# Cài đặt phần mềm
package_update: true
packages:
  - nginx
  - htop

# Khởi động dịch vụ
runcmd:
  - [ "systemctl", "enable", "nginx" ]
  - [ "systemctl", "start", "nginx" ]
```

### 2.2 Network-config

Network-config được sử dụng để cấu hình mạng cho máy ảo. Có hai cách chính để cấu hình mạng với cloud-init trong OpenStack:

#### 2.2.1 Netplan (Ubuntu)

Netplan là hệ thống cấu hình mạng được sử dụng trong Ubuntu. Network-config có thể được cấu hình theo định dạng YAML:

```yaml
network:
  version: 2
  ethernets:
    eth0:
      dhcp4: true
    eth1:
      addresses: ["10.0.0.100/24"]
      gateway4: 10.0.0.1
      nameservers:
        addresses: [8.8.8.8, 8.8.4.4]
```

#### 2.2.2 NetworkManager/ifupdown (CentOS/RHEL/Debian)

Đối với các hệ điều hành sử dụng NetworkManager hoặc ifupdown, network-config có thể được cấu hình dưới dạng script bash hoặc file định dạng riêng:

```bash
#!/bin/bash

# Cấu hình với NetworkManager
nmcli connection add type ethernet con-name eth0 ifname eth0 ip4 "10.0.0.100/24" gw4 "10.0.0.1" ipv4.dns "[EMAIL_ADDRESS] 8.8.4.4"

# Cấu hình với ifupdown
echo "auto eth0" >> /etc/network/interfaces
echo "iface eth0 inet static" >> /etc/network/interfaces
echo "    address 10.0.0.100" >> /etc/network/interfaces
echo "    netmask 255.255.255.0" >> /etc/network/interfaces
e



