OWASP Top 10 là một báo cáo được cập nhật thường xuyên về các nguy cơ bảo mật đối với bảo mật ứng dụng web, tập trung vào 10 rủi ro/lỗ hổng quan trọng nhất

Báo cáo được tổng hợp bởi một nhóm các chuyên gia bảo mật từ khắp nơi trên thế giới

# 1. Injection

**Injection attack** xảy ra khi dữ liệu không đáng tin cậy được gửi đến trình thông dịch mã (code interpreter) thông qua việc điền các form (biểu mẫu) hoặc một số dữ liệu khác gửi đến ứng dụng web

_Ví dụ_, kẻ tấn công có thể nhập SQL database code vào một biểu mẫu yêu cầu username ở dạng plaintext. Nếu việc điền các biểu mẫu đó không được bảo mật đúng cách, điều này sẽ dẫn đến việc SQL code đó được thực thi. Đây được gọi là một cuộc tấn công SQL injection

Các cuộc tấn công injection có thể được ngăn chặn bằng cách xác thực và / hoặc “khử trùng” dữ liệu do người dùng gửi. (Xác thực nghĩa là từ chối các dữ liệu đáng ngờ, trong khi “khử trùng” nghĩa là làm sạch các phần dữ liệu có vẻ đáng ngờ.)

Ngoài ra, quản trị viên cơ sở dữ liệu có thể thiết lập các biện pháp kiểm soát để giảm thiểu lượng thông tin bị lộ ra qua một cuộc tấn công injection

# 2. broken authentication

Các lỗ hổng trong hệ thống xác thực (login) có thể cho phép kẻ tấn công truy cập vào tài khoản người dùng và thậm chí có khả năng xâm nhập toàn bộ hệ thống bằng tài khoản quản trị viên

_Ví dụ_: kẻ tấn công có thể lấy một danh sách chứa hàng nghìn tổ hợp tên người dùng / mật khẩu đã biết có được trong một lần vi phạm dữ liệu và sử dụng tập lệnh để thử tất cả các tổ hợp đó trên hệ thống đăng nhập để xem có tổ hợp nào hoạt động không

Một số chiến lược để giảm thiểu lỗ hổng xác thực là sử dụng xác thực 2 yếu tố two-factor authentication (2FA) cũng như hạn chế hoặc trì hoãn các nỗ lực đăng nhập lặp lại bằng cách sử dụng giới hạn về số lần đăng nhập & thời gian giãn cách giữa các lần đăng nhập sai

# 3. Sensitive Data Exposure

Nếu các ứng dụng web không bảo vệ dữ liệu nhạy cảm như thông tin tài chính và mật khẩu, hacker có thể giành quyền truy cập vào dữ liệu đó và sử dụng cho các mục đích bất chính

Một phương pháp phổ biến để lấy cắp thông tin nhạy cảm là sử dụng một cuộc tấn công on-path attack

Nguy cơ lộ dữ liệu có thể được giảm thiểu bằng cách mã hóa (encypt) tất cả dữ liệu nhạy cảm cũng như vô hiệu \*cache của bất kỳ thông tin nhạy cảm nào. Ngoài ra, các nhà phát triển ứng dụng web nên cẩn thận để đảm bảo rằng họ không lưu trữ bất kỳ dữ liệu nhạy cảm nào một cách không cần thiết

Cache lưu trữ tạm thời dữ liệu để sử dụng lại.

_Ví dụ:_ trình duyệt web thường sẽ lưu vào bộ nhớ cache các trang web để nếu người dùng truy cập lại các trang đó trong một khoảng thời gian cố định, trình duyệt không phải tìm nạp các trang web đó từ đầu

# 4. XML External Entities (XEE)

Đây là một cuộc tấn công ứng dụng web bằng phân tích cú pháp đầu vào XML _ (parses XML_ input). Input này có thể tham chiếu đến một thực thể bên ngoài (external entity), đang cố gắng khai thác lỗ hổng trong trình phân tích cú pháp (parser)

External entity có thể là một đơn vị lưu trữ, chẳng hạn như ổ cứng. XML parser có thể bị lừa để gửi dữ liệu đến một thực thể bên ngoài trái phép và chuyển trực tiếp dữ liệu nhạy cảm cho kẻ tấn công

Các cách tốt nhất để ngăn chặn các cuộc tấn công XEE là để các ứng dụng web chấp nhận một loại dữ liệu ít phức tạp hơn, chẳng hạn như JSON \*\*, hoặc vô hiệu hóa việc sử dụng các thực thể bên ngoài trong một ứng dụng XML

\*XML hay Extensible Markup Language là một markup language nhằm mục đích cho phép cả người & máy đều có thể đọc hiểu được. Do tính phức tạp và lỗ hổng bảo mật, XML hiện đang bị loại bỏ dần trong nhiều ứng dụng web

\*\* JavaScript Object Notation (JSON) là một loại ký hiệu đơn giản được sử dụng để truyền dữ liệu qua internet. Mặc dù ban đầu được tạo cho JavaScript, JSON là ngôn ngữ có thể được thông dịch bởi nhiều ngôn ngữ lập trình khác nhau

# 5. Broken Access Control

Access Control hay kiểm soát truy cập đề cập đến một hệ thống kiểm soát quyền truy cập vào thông tin hoặc chức năng. Access Control chứa lỗ hổng cho phép kẻ tấn công bỏ qua ủy quyền (authorization) và thực hiện các tác vụ như thể là người dùng có đặc quyền, chẳng hạn như quản trị viên (admin)

_Ví dụ:_ một ứng dụng web có thể cho phép người dùng thay đổi tài khoản mà họ đã đăng nhập chỉ bằng cách thay đổi một phần của url mà không cần bất kỳ xác minh nào khác.

Kiểm soát truy cập có thể được bảo mật bằng cách đảm bảo rằng ứng dụng web sử dụng authorization tokens\* và đặt các kiểm soát chặt chẽ đối với các token này.

\*Nhiều dịch vụ cho phép sử dụng authorization tokens khi người dùng đăng nhập. Mọi yêu cầu đặc quyền mà người dùng đưa ra sẽ yêu cầu phải có authorization tokens. Đây là một cách an toàn để đảm bảo rằng đúng người dùng với đúng đặc quyền.

# 6. Security Misconfiguration

Security misconfiguration hay lỗi cấu hình sai bảo mật là lỗ hổng phổ biến nhất trong danh sách và thường là kết quả của việc sử dụng cấu hình mặc định hoặc thông báo hiển thị lỗi quá nhiều thông tin

_Ví dụ:_ một ứng dụng có thể hiển thị lỗi mô tả quá nhiều thông tin có thể tiết lộ các lỗ hổng trong ứng dụng. Điều này có thể được giảm thiểu bằng cách loại bỏ bất kỳ tính năng không sử dụng nào trong code và đảm bảo rằng các thông báo lỗi sẽ mang tính tổng quát chung chung hơn.

# 7. Cross-Site Scripting

Cross-Site Scripting xảy ra khi các ứng dụng web cho phép người dùng thêm code tùy chỉnh vào đường dẫn url hoặc vào một trang web mà những người dùng khác sẽ nhìn thấy

Lỗ hổng này có thể bị khai thác để chạy mã JavaScript độc hại (malicious JavaScript code) trên trình duyệt của nạn nhân.

_Ví dụ:_ kẻ tấn công có thể gửi email cho nạn nhân có vẻ là từ một ngân hàng đáng tin cậy, với một liên kết đến trang web của ngân hàng đó. Tuy nhiên, liên kết này có thể có một số mã JavaScript độc hại được gắn thẻ vào cuối url. Nếu trang web của ngân hàng không được bảo vệ thích hợp chống lại Cross-Site Scripting, thì mã độc hại đó sẽ được chạy trong trình duyệt web của nạn nhân khi họ nhấp vào liên kết.

Các chiến lược giảm thiểu tấn công Cross-Site Scripting bao gồm thoát các yêu cầu HTTP không đáng tin cậy cũng như xác thực và / hoặc loại bỏ các nội dung do người dùng thêm vào. Sử dụng các web development frameworks hiện đại như ReactJS và Ruby on Rails cũng cung cấp một số tính năng bảo vệ khỏi các cuộc tấn công Cross-Site Scripting.

# 8. Insecure Deserialization

Tấn công này bao gồm Serialization & Deserialization

- Serialization có nghĩa là lấy các đối tượng (object) từ mã ứng dụng (application code) và chuyển đổi chúng thành một định dạng có thể được sử dụng cho mục đích khác, chẳng hạn như lưu trữ dữ liệu vào đĩa hoặc phát trực tuyến dữ liệu đó

- Deserialization thì ngược lại với Serialization

Serialization giống như đóng gói đồ đạc vào các hộp trước khi chuyển đi, và deserialization giống như mở hộp và lắp ráp đồ đạc sau khi chuyển đi. Một cuộc tấn công deserialization giống như việc xáo trộn nội dung của các hộp trước khi chúng được giải nén trong quá trình di chuyển.

# 9. Sử dụng các thành phần có lỗ hổng đã biết

Nhiều nhà phát triển (developer) web hiện nay sử dụng các thành phần như thư viện (libraries) và framework trong các ứng dụng web của họ

Những thành phần này là những phần mềm giúp các nhà phát triển tránh công việc thừa và cung cấp chức năng cần thiết; ví dụ phổ biến bao gồm các framework front-end như React và các thư viện nhỏ hơn được sử dụng để thêm các biểu tượng chia sẻ hoặc a/b testing

Một số kẻ tấn công tìm kiếm các lỗ hổng trong các thành phần này mà sau đó chúng có thể sử dụng để điều phối các cuộc tấn công

Một số thành phần phổ biến hơn được sử dụng trên hàng trăm nghìn trang web; kẻ tấn công tìm thấy lỗ hổng bảo mật trong những thành phần này có thể khiến hàng trăm nghìn trang web bị khai thác.

Các nhà phát triển các thành phần này thường cung cấp các bản vá bảo mật và cập nhật để bổ sung các lỗ hổng đã biết, nhưng các nhà phát triển ứng dụng web không phải lúc nào cũng có các phiên bản được vá hoặc cập nhật mới nhất

Để giảm thiểu rủi ro khi chạy các thành phần có lỗ hổng đã biết, các nhà phát triển nên xóa các thành phần không sử dụng khỏi dự án, cũng như đảm bảo rằng đang nhận các thành phần từ một nguồn đáng tin cậy và đảm bảo chúng được cập nhật.

# 10. Kiểm tra log & giám sát không hiệu quả

Nhiều ứng dụng web không thực hiện đủ các bước để phát hiện vi phạm dữ liệu

Thời gian phát hiện trung bình cho một vi phạm là khoảng 200 ngày sau khi đã xảy ra

Điều này cho phép những kẻ tấn công có nhiều thời gian để gây ra thiệt hại trước khi có bất kỳ phản ứng nào

OWASP khuyến nghị rằng các nhà phát triển web nên thực hiện ghi log và giám sát (monitor) cũng như lên các kế hoạch ứng phó sự cố để đảm bảo rằng họ nhận thức được các cuộc tấn công vào các ứng dụng.
