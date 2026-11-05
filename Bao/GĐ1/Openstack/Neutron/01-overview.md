# Tổng quan về OpenStack Neutron

## 1. Khái niệm và các thành phần của Neutron

### 1.1 Khái niệm
**Neutron** là dịch vụ kết nối mạng (Networking Service) trong OpenStack. Nó cung cấp Networking-as-a-Service (NaaS), cho phép người dùng định nghĩa và quản lý các kiến trúc mạng ảo vô cùng phức tạp (như tạo mạng LAN ảo, Subnet ảo, Router ảo, Firewall, Load Balancer) thông qua các lời gọi API mà không cần phải chạy ra tủ rack cắm dây mạng thật.

### 1.2 Các thành phần chính
Kiến trúc của Neutron được xây dựng theo mô hình phân tán Controller-Agent:
- **neutron-server:** Nằm ở Node Controller. Tiếp nhận các request API từ người dùng. Nó chỉ phân tích lệnh, ghi vào Database, và điều phối công việc cho các agent bên dưới qua hàng đợi tin nhắn (RabbitMQ). Nó không thao tác trực tiếp với switch vật lý.
- **Database:** Lưu trạng thái mạng (Danh sách IP tĩnh, các port được cấp, quy định bảo mật).
- **Core Plugin (VD: ML2 - Modular Layer 2):** Khối lõi chịu trách nhiệm làm việc với các hệ thống Layer 2 (chuyển mạch). ML2 hỗ trợ gọi xuống nhiều loại driver vật lý khác nhau như LinuxBridge, OpenvSwitch (OVS).
- **Các Agent chạy trên các Node:**
  - **L2 Agent:** Chạy trên Compute Node, dùng để tạo cầu nối ảo (vSwitch), móc cáp mạng từ máy ảo gắn vào bridge ảo.
  - **L3 Agent:** Quản lý Router (định tuyến giữa các dải mạng) và SNAT/DNAT (Floating IP) để ra Internet.
  - **DHCP Agent:** Chạy dịch vụ cấp IP tự động (DNSMasq) cho máy ảo trong từng subnet.

## 2. Tìm hiểu các loại mạng (Provider vs Self-service)

Tùy vào nhu cầu vận hành, OpenStack chia mạng ảo ra làm 2 khái niệm chính:

- **Provider Network (Mạng nhà cung cấp):**
  - Do **Admin** tạo ra và quản lý.
  - Nó ánh xạ (map) 1:1 với một dải mạng vật lý có thật của Data Center (thường sử dụng kiến trúc VLAN vật lý hoặc Flat network).
  - **Đặc điểm:** Máy ảo gắn vào Provider Network sẽ nhận được địa chỉ IP thật của hạ tầng, có thể ping trực tiếp ra ngoài Internet hoặc mạng nội bộ công ty thông qua Gateway vật lý. Hiệu năng mạng cao nhưng tốn IP thật.

- **Self-service Network (Mạng tự phục vụ / Tenant Network):**
  - Do **Người dùng (End-user)** tự do tạo.
  - Nó là mạng "Overlay" hoàn toàn ảo (sử dụng công nghệ đường hầm VXLAN hoặc GRE).
  - **Đặc điểm:** Hai người dùng khác nhau có thể tạo ra các dải IP nội bộ trùng nhau (Ví dụ: 192.168.1.0/24) mà không hề bị xung đột. Tuy nhiên, các máy ảo này bị cô lập, không tự đi ra Internet được. Chúng bắt buộc phải tạo một Router Ảo, móc vào một Provider Network để xin "Floating IP" hoặc đi NAT ra ngoài.

## 3. Tìm hiểu file cấu hình của Neutron

Các cấu hình Neutron nằm rải rác ở `/etc/neutron/` với nhiều file nhỏ theo từng thành phần:
- `neutron.conf`: File cấu hình chung cho dịch vụ API. Chứa kết nối tới Database, RabbitMQ, thông tin xác thực Keystone.
- `plugins/ml2/ml2_conf.ini`: Cấu hình cho kiến trúc Layer 2. Định nghĩa các driver được phép dùng (VD: type_drivers = flat,vlan,vxlan) và dải ID cho VXLAN.
- `plugins/ml2/openvswitch_agent.ini`: (Trên Compute node) Chỉ cho L2 Agent biết card mạng vật lý thật tên là gì, sẽ phải gán vào cái cầu nối OVS nào.
- `l3_agent.ini`: Cấu hình cho L3 Agent, cấu hình giao diện kết nối External Router ảo ra ngoài mạng vật lý.
- `dhcp_agent.ini`: Khai báo driver (thường là dnsmasq) quản lý tiến trình DHCP.

## 4. Các câu lệnh thường dùng

*Lưu ý: Bạn phải source file cấu hình (openrc) trước.*
- Hiển thị danh sách mạng: `openstack network list`
- Tạo mạng mới: `openstack network create <tên_network>`
- Tạo subnet gắn vào mạng trên: `openstack subnet create --network <tên_network> --subnet-range <địa_chỉ/prefix> <tên_subnet>`
- Danh sách các port đang dùng: `openstack port list`
- Danh sách Router ảo: `openstack router list`
- Tạo một Floating IP (IP Public): `openstack floating ip create <tên_mạng_external>`

## 5. Đường đi gói tin với OpenvSwitch (Đông-Tây và Bắc-Nam)

**OpenvSwitch (OVS)** là xương sống mạng ảo phổ biến nhất. OVS sử dụng 3 bridge ảo chính: `br-int` (Integration - nơi gắn tất cả máy ảo), `br-tun` (Tunnel - xử lý đóng gói/giải mã VXLAN), và `br-ex` (External - nối ra mạng công cộng Internet).

- **Lưu lượng Đông - Tây (Giữa các máy ảo với nhau):**
  - *Cùng Compute Node:* VM1 -> `br-int` -> VM2. Gói tin không hề đi ra ngoài card vật lý.
  - *Khác Compute Node (Trong cùng một Subnet Self-Service):* VM1 -> `br-int` -> `br-tun`. Tại `br-tun`, gói tin Ethernet Layer 2 được đóng gói thêm vỏ VXLAN (chứa IP của Compute Node đích). Sau đó gói tin VXLAN bay qua mạng Data Center vật lý -> Sang Compute Node bên kia. Tại Node đích: Giải mã VXLAN ở `br-tun` -> Lấy gói tin L2 gốc chuyển sang `br-int` -> Đẩy vào VM2.

- **Lưu lượng Bắc - Nam (Từ máy ảo ra Internet hoặc ngược lại):**
  - Máy ảo ở Self-service subnet gửi gói tin -> `br-int` -> Đóng gói VXLAN ở `br-tun` -> Gửi sang Node có chạy L3 Agent (thường là Controller hoặc Network Node).
  - Tại Network Node: Gói tin qua `br-tun` (giải vỏ VXLAN) -> `br-int` -> Đi vào **Network Namespace** của chiếc Router ảo.
  - Bên trong không gian Namespace của Router: Tiến hành định tuyến và đổi IP (SNAT) thành IP Public.
  - Gói tin đẩy tiếp ra `br-ex` -> Qua card mạng vật lý -> Bay ra Internet.

## 6. Tìm hiểu về Namespace trong Neutron

**Network Namespace** là tính năng "cách ly mạng" kỳ diệu của hệ điều hành nhân Linux.
- OpenStack cho phép khái niệm mạng Self-service, nơi mà User A và User B có thể đặt dải IP nội bộ giống y hệt nhau (VD: 10.0.0.0/24).
- Nếu đưa các địa chỉ IP này vào bảng định tuyến (Routing table) chính thức của hệ điều hành máy chủ vật lý, máy chủ sẽ bị xung đột và không biết đẩy gói tin về phía User nào.
- Để giải quyết vấn đề này, Neutron ứng dụng **Network Namespace**. Khi bạn tạo một Router ảo hoặc bật DHCP cho một Subnet, Neutron sẽ sinh ra một không gian Linux ảo hoàn toàn độc lập (được gắn tên dạng `qrouter-xxx` hoặc `qdhcp-yyy`). Mỗi không gian này có một Bảng định tuyến (Routing table) riêng, card mạng riêng, luật Iptables riêng biệt. 
- Nhờ bị nhốt vào các hộp Namespace cách ly, IP của User A và User B trùng nhau vẫn có thể định tuyến bình thường mà không hề ảnh hưởng đến nhau.

## 7. Tìm hiểu về Security Group

Security Group (Nhóm bảo mật) là tính năng Tường lửa ảo (Firewall) bảo vệ ở cấp độ Card mạng (Port) của từng Máy Ảo, không phải tường lửa tổng của toàn mạng.
- Mặc định: Chặn mọi kết nối đi vào (Ingress Deny-all) và cho phép mọi kết nối đi ra (Egress Allow-all).
- Có trạng thái (Stateful): Nếu bạn cho phép ping đi, thì kết quả ping trả về sẽ tự động được nhận lại mà không cần mở rule chặn Ingress.
- Trong kiến trúc OVS hiện đại (sử dụng OVS Firewall Driver), các quy tắc Security Group (VD: Mở port 80, 22) được Neutron biên dịch trực tiếp thành các bộ luật "OpenFlow flow rules" nhúng thẳng vào `br-int`, giúp cản gói tin độc hại bằng sức mạnh của OpenvSwitch ngay khi gói tin vừa thò ra khỏi máy ảo, đem lại hiệu suất cực cao so với Iptables truyền thống.

## 8. Các đường Network trong mô hình OPS 3 Node (Vận hành)

Khi quy hoạch kiến trúc OpenStack chuẩn cho môi trường Lab/Sản xuất với 3 node (Controller, Compute, Block Storage), thiết kế mạng vật lý tối ưu sẽ chia làm 4 đường mạng (Network planes):
1. **Management Network (Mạng Quản trị API):** Dùng để các Node nói chuyện nội bộ với nhau (Nova gọi Keystone, Neutron chọc Database, ghi Log). Lưu lượng không lớn nhưng cần độ ổn định cực cao. Thường dùng dải mạng riêng.
2. **Provider / External Network (Mạng Public):** Nối Controller Node (hoặc Network Node) ra ngoài thế giới Internet. Mạng này cung cấp dải Floating IP.
3. **Data / Tunnel Network (Mạng Dữ liệu hầm):** Đường ống chuyên biệt cực kỳ quan trọng để xử lý lưu lượng VXLAN (máy ảo nói chuyện với máy ảo). Cần băng thông cực rộng (10G/40G) vì nó phải gánh lượng tải Data lớn nhất của hệ thống ảo hóa.
4. **Storage Network (Mạng Lưu trữ - Tùy chọn):** Đường mạng riêng kết nối các Compute Node với hệ thống đĩa Cinder (hoặc Ceph). Việc tách đường Storage giúp máy ảo đọc ghi ổ cứng không gây ảnh hưởng làm nghẽn đường hầm VXLAN ở trên.

## 9. Tìm hiểu VXLAN

**VXLAN (Virtual eXtensible Local Area Network)** là giao thức cốt lõi giúp các mạng "ảo" trong OpenStack có thể chui qua được mạng "vật lý" để đến các Node khác.
- **Tại sao lại cần?** Công nghệ VLAN cũ chỉ có 4096 ID (nghĩa là tạo tối đa 4000 mạng độc lập). Trong các đám mây Public Cloud hàng nghìn khách hàng, số lượng mạng lên đến hàng chục nghìn. VXLAN cung cấp độ lớn nhận dạng ID lên tới 16 triệu mạng ảo.
- **Cách hoạt động:** Nó sử dụng kỹ thuật đóng gói "MAC in UDP". Ethernet Frame (Gói tin Lớp 2) gốc chứa địa chỉ MAC của máy ảo sẽ bị bọc lại thành một cục Data, và nhét vào thân một gói tin UDP (Lớp 4). Lớp vỏ UDP bên ngoài mang IP Nguồn/Đích chính là IP vật lý của các máy chủ Compute.
- Các Switch mạng vật lý thật của Data Center chỉ nhìn thấy các gói tin UDP chạy giữa các máy chủ vật lý, chứ không hề biết gì về các địa chỉ IP nội bộ của các máy ảo giấu bên trong.

## 10. Tìm hiểu QoS (Quality of Service) trong Neutron

QoS trong Neutron được tạo ra để ngăn chặn tình trạng "người hàng xóm ồn ào" (Noisy Neighbor) - một máy ảo bị nghẽn mạng gây sập toàn bộ đường truyền băng thông của máy chủ vật lý.
- Tính năng này cho phép quản trị viên đặt ra các chính sách (Policy) giới hạn lưu lượng mạng cho từng Port cụ thể hoặc áp dụng cho cả Subnet.
- **Các luật QoS phổ biến:**
  - **Bandwidth Limit (Giới hạn băng thông):** Áp đặt tốc độ tải lên/tải xuống tối đa (ví dụ: giới hạn 50 Mbps cho máy ảo A).
  - **DSCP Marking:** Nhãn ưu tiên gói tin (để hệ thống mạng vật lý đọc và ưu tiên trước, VD ưu tiên các gói VoIP).
- **Cơ chế thực thi:** Dựa vào các driver (như OVS hoặc LinuxBridge). Chúng sẽ sử dụng tính năng kiểm soát luồng `tc` (Traffic Control) trên nhân Linux hoặc cơ chế Token Bucket của OpenvSwitch để vứt bỏ các gói tin nếu máy ảo đó truyền dữ liệu vượt quá tốc độ Max Kbps đã được khai báo.
