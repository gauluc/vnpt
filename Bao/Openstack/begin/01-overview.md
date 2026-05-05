# Cloud Computing & OpenStack Overview

## 1. Cloud Computing là gì?

Điện toán đám mây (Cloud Computing) là việc cung cấp các dịch vụ điện toán (bao gồm máy chủ, lưu trữ, cơ sở dữ liệu, mạng, phần mềm, phân tích, và trí tuệ nhân tạo) qua Internet ("đám mây") để mang lại sự đổi mới nhanh hơn, tài nguyên linh hoạt và lợi thế về quy mô (economies of scale). 

Thay vì phải tự mua, sở hữu, và bảo trì các trung tâm dữ liệu hay máy chủ vật lý, bạn có thể thuê các dịch vụ điện toán từ một nhà cung cấp (ví dụ: AWS, Google Cloud, Microsoft Azure) hoặc tự xây dựng hệ thống Private Cloud của riêng mình và chỉ trả tiền cho những gì bạn sử dụng.

## 2. Mô hình 5-4-3 trong Cloud Computing

Mô hình 5-4-3 là tiêu chuẩn được Viện Tiêu chuẩn và Công nghệ Quốc gia Hoa Kỳ (NIST) định nghĩa để mô tả điện toán đám mây. Nó bao gồm 5 đặc điểm cơ bản, 4 mô hình triển khai, và 3 mô hình dịch vụ:

### 5 Đặc điểm cơ bản (5 Essential Characteristics)
1. **On-demand self-service (Tự phục vụ theo nhu cầu):** Người dùng có thể tự động cung cấp tài nguyên máy tính (như thời gian sử dụng máy chủ và dung lượng lưu trữ mạng) khi cần thiết mà không cần tương tác trực tiếp với nhà cung cấp dịch vụ.
2. **Broad network access (Truy cập mạng rộng rãi):** Khả năng truy cập các dịch vụ đám mây từ mọi nơi, sử dụng nhiều loại thiết bị khác nhau (điện thoại thông minh, máy tính bảng, laptop, máy trạm).
3. **Resource pooling (Dùng chung tài nguyên):** Các tài nguyên điện toán của nhà cung cấp được gom chung lại để phục vụ nhiều người tiêu dùng (multi-tenant model), với các tài nguyên vật lý và ảo hóa được cấp phát và thu hồi động tùy theo nhu cầu.
4. **Rapid elasticity (Co giãn nhanh chóng):** Tài nguyên có thể được cấp phát (scale up/out) và thu hồi (scale down/in) một cách nhanh chóng và tự động trong một số trường hợp để đáp ứng với biến động nhu cầu.
5. **Measured service (Dịch vụ được đo lường):** Hệ thống đám mây tự động kiểm soát và tối ưu hóa tài nguyên bằng cách sử dụng khả năng đo lường ở một mức độ trừu tượng thích hợp với loại dịch vụ. Cung cấp tính minh bạch (pay-as-you-go).

### 4 Mô hình triển khai (4 Deployment Models)
1. **Public Cloud (Đám mây công cộng):** Cơ sở hạ tầng đám mây được mở cho sử dụng công cộng (ví dụ: AWS, Azure).
2. **Private Cloud (Đám mây riêng):** Đám mây được vận hành dành riêng cho một tổ chức duy nhất (có thể tự quản lý hoặc do bên thứ 3 quản lý).
3. **Community Cloud (Đám mây cộng đồng):** Cơ sở hạ tầng đám mây được chia sẻ bởi một vài tổ chức có chung mối quan tâm (như bảo mật, chính sách).
4. **Hybrid Cloud (Đám mây lai):** Sự kết hợp của hai hoặc nhiều cơ sở hạ tầng đám mây khác nhau (Public, Private, Community) được liên kết bằng các công nghệ chuẩn.

### 3 Mô hình dịch vụ (3 Service Models)
1. **IaaS (Infrastructure as a Service - Hạ tầng như một dịch vụ):** Cung cấp các tài nguyên điện toán cơ bản như máy ảo, mạng, lưu trữ (ví dụ: AWS EC2, OpenStack).
2. **PaaS (Platform as a Service - Nền tảng như một dịch vụ):** Cung cấp môi trường (nền tảng) cho các nhà phát triển để tạo, kiểm thử, và triển khai phần mềm mà không cần quản lý hạ tầng bên dưới (ví dụ: Heroku, Google App Engine).
3. **SaaS (Software as a Service - Phần mềm như một dịch vụ):** Cung cấp phần mềm hoàn thiện qua Internet, người dùng chỉ cần sử dụng (ví dụ: Gmail, Office 365, Salesforce).

---

## 3. OpenStack là gì? Kiến trúc cơ bản

### OpenStack là gì?
OpenStack là một nền tảng phần mềm mã nguồn mở để xây dựng và quản lý các môi trường **điện toán đám mây** (Cloud Computing), thường được sử dụng nhiều nhất để triển khai **Private Cloud** hoặc Public Cloud. 

Nó cung cấp cơ sở hạ tầng dưới dạng dịch vụ (IaaS), cho phép người dùng triển khai máy ảo (VM), mạng ảo (SDN), khối lưu trữ ảo (Block Storage) và các dịch vụ khác thông qua một bảng điều khiển (Dashboard - Horizon) hoặc API/CLI.

### Kiến trúc cơ bản của OpenStack
OpenStack không phải là một phần mềm đơn lẻ mà là một tập hợp các dự án (hoặc dịch vụ) phối hợp với nhau. Dưới đây là các thành phần cốt lõi tạo nên kiến trúc OpenStack:

1. **Keystone (Identity Service):** Cung cấp dịch vụ xác thực, phân quyền và danh mục dịch vụ cho toàn bộ hệ thống OpenStack. Đây là "người gác cổng".
2. **Nova (Compute Service):** Quản lý và cung cấp máy ảo (VM) theo nhu cầu. Nova tương tác với các hypervisor (như KVM, VMware) để tạo và quản lý máy ảo.
3. **Neutron (Networking Service):** Cung cấp khả năng kết nối mạng ảo (SDN) giữa các thiết bị được quản lý bởi OpenStack (như gán IP, tạo mạng ảo, router ảo, firewall).
4. **Cinder (Block Storage Service):** Cung cấp hệ thống lưu trữ dạng block (ổ cứng ảo) cho các máy ảo của Nova sử dụng.
5. **Glance (Image Service):** Lưu trữ, quản lý và truy xuất các disk image (như ISO, QCOW2) để làm khuôn mẫu khởi tạo các máy ảo.
6. **Swift (Object Storage Service):** Hệ thống lưu trữ đối tượng (tương tự như Amazon S3), dùng để lưu trữ dữ liệu phi cấu trúc (file tĩnh, backup).
7. **Horizon (Dashboard):** Giao diện Web GUI để người dùng và quản trị viên quản lý, cấp phát, và theo dõi tài nguyên của OpenStack.

*Ngoài các thành phần cốt lõi này, OpenStack còn rất nhiều dự án khác như Heat (Orchestration), Ceilometer (Telemetry), Ironic (Bare-metal provisioning),... để bổ sung thêm các tính năng cao cấp.*
