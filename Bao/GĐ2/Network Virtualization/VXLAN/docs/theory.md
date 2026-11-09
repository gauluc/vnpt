# 1. VXLAN – Virtual Extensible LAN

**VXLAN** là một công nghệ ảo hóa mạng theo mô hình Overlay Network (mạng xếp chồng), cho phép đóng gói (encapsulate) các gói tin Ethernet Layer 2 (MAC-in-UDP) và truyền tải chúng qua hạ tầng mạng Layer 3 (IP)

**VXLAN** giải quyết một số hạn chế của **VLAN** truyền thống

| VLAN | VXLAN |
| :--- | :--- |
| 12-bit VLAN ID | 24-bit VNI |
| Tối đa 4094 VLAN ID sử dụng thông thường | Khoảng 16,7 triệu VNI |
| Phụ thuộc nhiều vào Layer 2 | Chạy trên Layer 3 Underlay |
| Khó mở rộng Layer 2 giữa các Data Center | Hỗ trợ mở rộng mạng logic linh hoạt |

### Các thành phần quan trọng

1. VTEP (VXLAN Tunnel Endpoint)
    - Điểm thực hiện encapsulation và decapsulation
    - Có thể nằm trên Leaf Switch hoặc hypervisor
    - Tạo và kết thúc VXLAN tunnel
2. VNI (VXLAN Network Identifier)
    - Định danh một VXLAN segment
    - Tương tự VLAN ID nhưng có không gian định danh lớn hơn
3. Underlay Network
    - Mạng vật lý IP kết nối các VTEP
    - Thường sử dụng OSPF hoặc BGP
    - Chịu trách nhiệm vận chuyển packet
4. Overlay Network
    - Mạng logic được tạo bằng VXLAN
    - Cho phép các VM thuộc cùng một mạng logic giao tiếp qua hạ tầng Layer 3

### Cơ chế hoạt động:

**VTEP (VXLAN Tunnel End Point):** Là thực thể thực hiện nhiệm vụ đóng gói và giải đóng gói gói tin VXLAN. VTEP có thể là một switch vật lý (Hardware VTEP) hoặc một switch ảo chạy trên Hypervisor (như Open vSwitch)

**Quy trình:** Khi máy ảo gửi gói tin Layer 2 đi ra ngoài, VTEP nguồn sẽ bắt gói tin này, bọc thêm Header VXLAN (chứa VNI), Header UDP (Port đích mặc định là 4789), Header IP (IP nguồn/đích của 2 VTEP), và Header Ethernet ngoài cùng. Gói tin này được truyền đi như một gói tin IP bình thường qua mạng vật lý (Underlay Network). Khi tới VTEP đích, Header ngoài cùng và VXLAN sẽ bị bóc tách để khôi phục lại gói tin Layer 2 nguyên bản và chuyển tiếp đến máy ảo đích

**Ví dụ:**

- VM-A nằm trên Server A, thuộc Leaf 1
- VM-B nằm trên Server B, thuộc Leaf 2
- Hai VM cùng VNI 10010
- Hai Leaf đóng vai trò VTEP

Quá trình truyền packet:

1. VM-A gửi Ethernet frame.
2. VTEP tại Leaf 1 nhận frame.
3. Leaf 1 đóng gói Ethernet frame vào VXLAN.
4. VXLAN packet được truyền qua IP Underlay.
5. Spine chuyển tiếp packet dựa trên địa chỉ IP đích VTEP.
6. Leaf 2 nhận packet và decapsulation.
7. Ethernet frame được chuyển đến VM-B.

## Flood and Learn (Multicast hoặc HER)

### Multicast-based BUM Flooding

```
VTEP 1 map VNI 1000 → Multicast group 239.1.1.1
VTEP 2, VTEP 3 join cùng multicast group

Khi VM-A gửi ARP broadcast:
VTEP 1 encapsulate → Dst IP = 239.1.1.1 (multicast)
VTEP 2, 3 nhận → decapsulate → forward vào VNI 1000
VTEP 2, 3 học: MAC-A → VTEP 1
```

**Yêu cầu**: PIM multicast trên underlay.

### Head-End Replication (HER / Unicast Ingress Replication)

```
VTEP 1 có danh sách tất cả VTEP trong VNI 1000 (tĩnh hoặc từ control plane)

Khi VM-A gửi BUM:
VTEP 1 copy packet → gửi unicast đến VTEP 2
VTEP 1 copy packet → gửi unicast đến VTEP 3
```

**Ưu điểm**: Không cần multicast trên underlay.
**Nhược điểm**: Tốn CPU tại ingress VTEP, nhiều unicast copies.


# 2. BGP EVPN – Control Plane của VXLAN

BGP EVPN là control plane cho:
- VXLAN (datacenter overlay)
- MPLS L2VPN (replacing VPLS)
- Multi-homing (ESI)

VXLAN và EVPN có vai trò khác nhau:
- VXLAN: Data Plane – đóng gói và vận chuyển traffic
- EVPN: Control Plane – phân phối thông tin MAC/IP và reachability

Một số EVPN Route Type quan trọng:

- Type 1: Ethernet Auto-Discovery
- Type 2: MAC/IP Advertisement
- Type 3: Inclusive Multicast Ethernet Tag
- Type 4: Ethernet Segment
- Type 5: IP Prefix Advertisement

Trong thực tế, BGP EVPN giúp các VTEP học thông tin endpoint mà không cần phụ thuộc vào cơ chế flood-and-learn truyền thống

