# BÁO CÁO KIỂM THỬ XÂM NHẬP

## 1. Tổng quan

Bài kiểm thử được thực hiện trên một máy ảo VulnHub trong môi trường lab nhằm xác định các lỗ hổng bảo mật và đánh giá khả năng khai thác của hệ thống.

Quá trình kiểm thử bao gồm các giai đoạn:

- Thu thập thông tin và quét cổng.
- Enumeration các dịch vụ Web.
- Tìm kiếm các tài nguyên và endpoint ẩn.
- Phân tích request bằng Burp Suite.
- Xác định lỗ hổng Arbitrary File Write.
- Khai thác lỗ hổng để thực thi mã PHP và nhận Reverse Shell.
- Thu thập User Flag.
- Enumeration hệ thống sau khi có quyền truy cập.
- Xác định lỗ hổng Kernel CVE-2022-0847.
- Khai thác Dirty Pipe để leo thang đặc quyền lên `root`.

---

# 2. Quét cổng và dịch vụ

Đầu tiên, sử dụng Nmap để quét các cổng đang mở trên địa chỉ IP của máy mục tiêu.

![Nmap](img/nmap.png?raw=true)

Kết quả quét cho thấy có 3 cổng đang mở:

| Cổng | Dịch vụ | Nhận xét |
|---|---|---|
| 22/tcp | SSH | Dịch vụ SSH |
| 80/tcp | HTTP | Ứng dụng Web |
| 81/tcp | HTTP | Yêu cầu xác thực |

Đáng chú ý, dịch vụ HTTP trên cổng `81` trả về HTTP status code `401 Unauthorized`, cho thấy ứng dụng yêu cầu xác thực.

Tiến hành truy cập hai dịch vụ Web trên cổng `80` và `81` để thu thập thêm thông tin.

![Port 80](img/port80.png?raw=true)

![Port 81](img/port81.png?raw=true)

Dịch vụ trên cổng `81` yêu cầu đăng nhập. Do chưa xác định được thông tin xác thực tại thời điểm này, quá trình kiểm thử tiếp tục tập trung vào ứng dụng Web trên cổng `80`.

---

# 3. Web Enumeration

Tiến hành sử dụng Gobuster để tìm kiếm các thư mục và file ẩn trên ứng dụng Web.

![Gobuster](img/gobuster.png?raw=true)

Kết quả cho thấy ứng dụng có khá nhiều đường dẫn đáng chú ý.

Một trong những tài nguyên được phát hiện là:

```text
robots.txt
```

Truy cập vào `robots.txt` nhận được nội dung:

```text
There's no white rabbit here. Keep searching!
```

Nội dung này không cung cấp thông tin trực tiếp có thể khai thác, vì vậy tiếp tục kiểm tra các tài nguyên khác được Gobuster phát hiện.

---

# 4. Phát hiện chức năng Graffiti

Trong quá trình enumeration, endpoint:

```text
graffity.php
```

được phát hiện.

![Graffiti](img/graffity.png?raw=true)

Endpoint này cung cấp chức năng gửi các message lên một graffiti wall.

Ngoài ra, file:

```text
graffity.txt
```

cũng được phát hiện.

Nội dung của file:

```text
Mouse here - welcome to the Nebby!

Make sure not to tell Morpheus about this graffiti wall.
It's just here to let us blow off some steam.
```

Việc ứng dụng có chức năng ghi nội dung vào file cho thấy cần kiểm tra cách ứng dụng xử lý các tham số liên quan đến filesystem.

---

# 5. Phân tích Request bằng Burp Suite

Tiếp theo, sử dụng Burp Suite để bắt và phân tích request được gửi từ chức năng graffiti.

![Burp Request](img/burptxt.png?raw=true)

Request chứa hai tham số đáng chú ý:

```text
message
file
```

Tiến hành thay đổi giá trị của tham số `file` thành:

```text
graffity.txt
```

Kết quả cho phép truy cập nội dung của file `graffity.txt` đã được phát hiện trước đó.

Điều này cho thấy tham số `file` có khả năng được ứng dụng sử dụng trực tiếp để xác định file cần thao tác.

Tiếp tục thử nghiệm bằng cách thay đổi giá trị thành một file PHP.

![Burp PHP](img/burpphp.png?raw=true)

Kết quả thu được nội dung source code của `graffity.php`:

```php
$file="graffiti.txt";
if($_SERVER['REQUEST_METHOD'] == 'POST') {
    if(isset($_POST['file'])) {
        $file=$_POST['file'];
    }

    if (isset($_POST['message'])) {
        $handle = fopen($file, 'a+') or die('Cannot open file: ' . $file);
        fwrite($handle, $_POST['message']);
        fwrite($handle, "\n");
        fclose($file);
    }
}
```

---

# 6. Phân tích lỗ hổng Arbitrary File Write

## 6.1. Phân tích mã nguồn

Đoạn code cho thấy giá trị của biến `$file` có thể được thay đổi trực tiếp thông qua tham số `file` do người dùng kiểm soát:

```php
if(isset($_POST['file'])) {
    $file=$_POST['file'];
}
```

Sau đó, giá trị này được sử dụng trực tiếp trong hàm:

```php
fopen($file, 'a+')
```

Nội dung của tham số `message` cũng được kiểm soát bởi người dùng và được ghi vào file thông qua:

```php
fwrite($handle, $_POST['message']);
```

Ứng dụng không thực hiện cơ chế kiểm tra hoặc giới hạn đường dẫn file mà người dùng có thể cung cấp.

Do đó, attacker có khả năng kiểm soát file đích và nội dung được ghi vào file, với điều kiện PHP process có quyền ghi vào vị trí đó.

Đây là lỗ hổng **Arbitrary File Write**.

Nếu attacker có thể ghi một file PHP vào thư mục Web và Web server cho phép thực thi file PHP đó, lỗ hổng có thể được nâng cấp thành khả năng **Remote Code Execution**.

---

# 7. Khai thác Arbitrary File Write

Dựa trên phân tích ở trên, tiến hành khai thác chức năng ghi file để tạo một file PHP do attacker kiểm soát.

Tham số:

```text
file = shell.php
```

được sử dụng để chỉ định file cần tạo.

Tham số `message` được sử dụng để ghi mã PHP thực hiện reverse shell.

Request được chỉnh sửa bằng Burp Suite:

![Burp Shell](img/burpshell.png?raw=true)

Sau khi gửi request, file `shell.php` được tạo trên máy chủ.

Tiếp theo, truy cập file này để thực thi payload.

Kết quả nhận được một Reverse Shell từ máy chủ:

![Reverse Shell](img/reverseshell.png?raw=true)

Như vậy, lỗ hổng Arbitrary File Write đã được khai thác thành công để đạt quyền truy cập ban đầu vào hệ thống.

---

# 8. Initial Access

Sau khi Reverse Shell được thiết lập thành công, attacker đã có quyền truy cập vào hệ thống với tài khoản có đặc quyền hạn chế.

Từ thời điểm này, quá trình kiểm thử chuyển sang giai đoạn **Local Enumeration** nhằm tìm kiếm các thông tin và cơ chế có thể sử dụng để leo thang đặc quyền.

---

# 9. Thu thập User Flag

Sau khi có quyền truy cập vào hệ thống, tiến hành tìm kiếm file chứa User Flag.

File:

```text
FLAG.txt
```

được phát hiện và có thể đọc bằng tài khoản hiện tại.

![User Flag](img/user_flag.png?raw=true)

Việc đọc thành công User Flag xác nhận rằng quá trình khai thác Web đã giúp attacker đạt được quyền truy cập vào hệ thống.

---

# 10. Local Privilege Escalation Enumeration

Sau khi có Initial Access, sử dụng `linpeas.sh` để rà soát hệ thống và tìm kiếm các vector có thể sử dụng cho Local Privilege Escalation.

![CVE](img/cve.png?raw=true)

Kết quả quét cho thấy hệ thống có một số candidate Kernel vulnerability.

Trong số đó, **CVE-2022-0847 – Dirty Pipe** là một lỗ hổng đáng chú ý và được lựa chọn để kiểm tra khả năng khai thác.

---

# 11. CVE-2022-0847 – Dirty Pipe

**Tên:** Dirty Pipe  
**CVE:** CVE-2022-0847  
**Loại:** Local Privilege Escalation  
**Mức độ:** Critical

CVE-2022-0847 là một lỗ hổng trong Linux Kernel liên quan đến cơ chế xử lý `pipe_buffer`.

Lỗi xuất phát từ việc thành phần `flags` trong cấu trúc `pipe_buffer` tại một số đường xử lý như `copy_page_to_iter_pipe` và `push_pipe` không được khởi tạo đúng cách, dẫn tới việc có thể giữ lại các giá trị cũ.

Trong điều kiện khai thác phù hợp, một người dùng không có đặc quyền có thể lợi dụng lỗi này để tác động đến dữ liệu trong page cache của các file chỉ đọc.

Điều này có thể cho phép attacker sửa đổi nội dung của một số file mà tài khoản hiện tại thông thường không có quyền ghi.

Nếu các file quan trọng của hệ thống bị tác động, lỗ hổng có thể được sử dụng để thực hiện **Local Privilege Escalation** và đạt quyền `root`.

---

# 12. Khai thác CVE-2022-0847

Để kiểm tra khả năng khai thác, sử dụng PoC cho CVE-2022-0847:

```text
https://github.com/Al1ex/CVE-2022-0847
```

PoC được tải xuống và chuyển lên máy mục tiêu.

Sau đó cấp quyền thực thi cho file `exp` và tiến hành chạy PoC.

![Root](img/root.png?raw=true)

Kết quả khai thác thành công, cho phép attacker leo thang đặc quyền lên `root`.

Quá trình khai thác đã hoàn tất mục tiêu của bài lab, từ quyền truy cập ban đầu thông qua ứng dụng Web đến quyền quản trị cao nhất trên hệ thống.

---

# 13. Chuỗi tấn công

Toàn bộ quá trình khai thác có thể được mô tả như sau:

```text
                 Target
                   │
                   ▼
             Nmap Scan
                   │
                   ▼
        ┌──────────┼──────────┐
        │          │          │
       22         80         81
       SSH        HTTP       HTTP
                              │
                         Authentication
                              │
                              ▼
                         HTTP :80
                              │
                              ▼
                       Gobuster Scan
                              │
                              ▼
                        graffity.php
                              │
                              ▼
                       Burp Suite
                              │
                              ▼
                    Source Code Analysis
                              │
                              ▼
                    Arbitrary File Write
                              │
                              ▼
                        shell.php
                              │
                              ▼
                      Reverse Shell
                              │
                              ▼
                       Initial Access
                              │
                              ▼
                         linpeas.sh
                              │
                              ▼
                      Kernel Enumeration
                              │
                              ▼
                       CVE-2022-0847
                              │
                              ▼
                       Root Privilege
```

---

# 14. Tổng hợp lỗ hổng

| ID | Lỗ hổng | Mức độ | Trạng thái |
|---|---|---|---|
| VULN-01 | Arbitrary File Write tại `graffity.php` | High | Đã khai thác |
| VULN-02 | CVE-2022-0847 – Dirty Pipe | Critical | Đã khai thác |

Hai lỗ hổng có thể được kết hợp thành một chuỗi tấn công:

```text
Arbitrary File Write
        ↓
PHP Code Execution
        ↓
Reverse Shell
        ↓
Initial Access
        ↓
Kernel Exploitation
        ↓
Root
```