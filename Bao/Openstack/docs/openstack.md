# 1. OpenStack là gì ?

OpenStack là nền tảng mã nguồn mở miễn phí cung cấp một Frame để xây dựng và quản lý cơ sở hạ tầng public cloud và private cloud

OpenStack cung cấp cơ sở hạ tầng như một dịch vụ, cấu hình và quản lý một lượng lớn tài nguyên điện toán, storage và network

Các tài nguyên này bao gồm : metal hardware, máy ảo (VM)
=> được quản lý thông qua giao diện lập trình ứng dụng (API) và bảng thông tin OpenStack

Các tổ chức và nhà cung cấp dịch vụ có thể triển khai OpenStack tại chỗ (tạo private cloud trong data center), OpenStack trong cloud để kích hoạt hoặc cung cấp năng lượng cho các nền tảng đám mây và các hệ thống network

# 2. OpenStack dùng để làm gì ?

![image](../image/openstack-ung-dung.png)

Để tạo môi trường điện toán đám mây, các tổ chức thường xây dựng cơ sở hạ tầng ảo hóa của mình bằng cách sử dụng các hệ thống nổi tiếng như VMware vSphere, Microsoft Hyper-V hoặc KVM

Tuy nhiên, điện toán đám mây ngoài cung cấp ảo hóa public cloud và private cloud còn cung cấp khả năng tự động hóa vòng đời tự phục vụ của người dùng, báo cáo chi phí và thanh toán, đồng bộ hóa và các khả năng khác

Cài đặt phần mềm OpenStack trên môi trường ảo tạo ra một cloud operating system

Các công ty có thể sử dụng nó để tổ chức, đặt cấu hình và quản lý các nhóm tài nguyên mạng, storage và network resources khác nhau, trong khi quản trị viên CNTT thường định cấu hình và quản lý tài nguyên trong môi trường ảo truyền thống

Cơ sở hạ tầng ảo hóa được xây dựng thông qua OpenStack có rất nhiều tác dụng, bao gồm tạo web hosting, Server chứa các dự án dữ liệu lớn và các phần mềm dưới dạng dịch vụ

OpenStack cạnh tranh trực tiếp với các nền tảng đám mây mã nguồn mở khác, ví dụ như Eucalyptus và Apache CloudStack, một số người cũng coi nó như một giải pháp thay thế cho các nền tảng đám mây miễn phí như Amazon Web Services hoặc Microsoft Azure, với một số công ty về hosting, cloud nhỏ hơn sử dụng OpenStack làm nền tảng gốc dịch vụ của mình

# 3. OpenStack hoạt động như thế nào

![image](../image/openstack-hoat-dong.png)

OpenStack không phải là một ứng dụng theo nghĩa truyền thống, nhưng nó là một nền tảng được tạo thành từ hàng chục thành phần độc lập được gọi là projects, hoạt động cùng nhau thông qua các API

Các tổ chức chỉ có thể cài đặt các thành phần được chọn để tạo ra các tính năng và chức năng mong muốn trong môi trường đám mây.

OpenStack cũng dựa vào hai công nghệ nền tảng: hệ điều hành cơ bản (như Linux) và nền tảng ảo hóa (như VMware hoặc Citrix), một hệ điều hành xử lý các hướng dẫn và dữ liệu được trao đổi từ OpenStack, trong khi công cụ ảo hóa quản lý các tài nguyên phần cứng ảo được sử dụng bởi projects OpenStack.

Sau khi hệ điều hành, nền tảng ảo hóa và các thành phần OpenStack được triển khai và cấu hình đúng cách, quản trị viên có thể định cấu hình và quản lý các tài nguyên được tạo sẵn mà ứng dụng cần. Các hành động và yêu cầu được thực hiện thông qua trang tổng quan tạo ra một loạt các lệnh gọi API được xác thực bởi dịch vụ bảo mật và được chuyển đến thành phần đích. thực hiện các nhiệm vụ liên quan.

Ví dụ: administrator đăng nhập vào OpenStack và quản lý môi trường đám mây thông qua dashboard. Administrator có thể tạo mới và kết nối các phiên bản điện toán và storage mới cũng như đặt cấu hình network behaviors

Administrator cũng có thể kết nối với các dịch vụ khác, chẳng hạn như giám sát hiệu năng phiên bản được cung cấp và sử dụng tính phí và bồi hoàn tài nguyên cho dung lượng lưu trữ.

Phạm vi rộng của nền tảng OpenStack và số lượng các thành phần được kết nối với nhau có thể gây nhầm lẫn và khó khăn. Hầu hết người dùng OpenStack bắt đầu với một số lượng nhỏ các yếu tố cơ bản và dần dần triển khai các phần tử khác. Theo thời gian, để xây dựng các khả năng hoạt động và kinh doanh của đám mây của họ.
