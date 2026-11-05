# 1. Container

**Container không phải máy ảo**

Container là một công nghệ ảo hóa cấp hệ điều hành, dùng để đóng gói mã nguồn ứng dụng cùng toàn bộ các thư viện, công cụ và cấu hình phụ thuộc thành một gói duy nhất, độc lập và nhẹ.
Nó giúp ứng dụng chạy ổn định, đồng nhất trên mọi môi trường (máy tính cá nhân, server, đám mây) mà không sợ xung đột hệ thống

-> App + thư viện + config → đóng thành 1 “hộp” (container)
-> Chạy ở đâu cũng giống nhau (laptop, server, cloud…)

Container giúp giải quyết

- vấn đề môi trường riêng
- deploy cực nhanh -> tăng tốc độ phát triển
- tối ưu tài nguyên

Nó dùng kernel của OS host (Linux) nhưng tách biệt bằng 2 công nghệ chính
