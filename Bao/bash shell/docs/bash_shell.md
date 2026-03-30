# 1. Tạo bash

tạo `sudo nano hello.sh`

nhập nội dung `#!/bin/bash ...` _#!/bin/bash gọi là shebang (chỉ định shell dùng để chạy)_

cấp quyền chạy `sudo chmod +x hello.sh`

chạy `./hello.sh`

# 2. câu lệnh trong bash

biến `name="bao"`, dùng `$name` _name="bao" viết liền_

nhập dữ liệu `read`

in ra `echo`

tìm kiếm `grep`

xử lý text `awk`

chỉnh sửa text `sed`

cắt chuối `cut`

thay thế ký tự `tr`

so sánh `-eq` = , `-ne` ≠ , `-gt` > , `-lt'` <

**vòng lặp**

```bash
for i in 1 2 3 4 5
do
echo "So: $i"
done
```

```bash
i=1
while [ $i -le 5 ]
do
echo $1
((i++))
done
```

**hàm**

```bash
hello() {
echo "Hello $1"
}
hello Bao
```

**backup đơn giản**

```bash
#!/bin/bash

src="/home"
dest="/home/le/backup"
mkdir -p "$dest"
tar -czf $dest/backup_$(date +%F).tar.gz $src

echo "Backup done!"
```

![image](vnpt/Bao/bash shell/image/filebackup.png)
