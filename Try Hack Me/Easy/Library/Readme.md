- Chúng ta sẽ khởi động máy ảo và quét các port của IP bằng nmap
``nmap -sV 10.49.163.225``
- ![img](img/nmap.png?raw=true)
- Từ kết quả của nmap có thể thấy 2 port là 22 và 80 đang được mở
- Chúng ta sẽ thử mở IP này ở port 80
- ![img](img/web.png?raw=true)
- Thấy tên một người dùng là 'meliodas' có thể là một user trong ssh
- Dùng gobuster để tìm những file ẩn
``gobuster dir -u http://10.49.163.255 -w /usr/share/dirb/wordlists/common.txt``
- ![img](img/gobuster.png?raw=true)
- Thấy một file ẩn đáng nghi là 'robots.txt'. Hãy thử truy cập vào nó xem
- ![img](img/robots.png?raw=true)
- Đây có thể là một gợi ý về việc bruteforce mật khẩu khi truy cập vào ssh với tên người dùng là meliodas và file rockyou trong kali linux
- Chúng ta sẽ dùng hydra để crack mật khẩu
``hydra -l meliodas -P /usr/share/wordlists/rockyou.txt ssh://10.49.163.225``
- ![img](img/hydra.png?raw=true)
- Đăng nhập vào ssh với tên người dùng và mật khẩu này
- ![img](img/ssh.png?raw=true)
- Chúng ta đã đăng nhập vào được với tên người dùng là meliodas
- ![img](img/user.png?raw=true)
- Chúng ta sẽ thử dùng ``sudo -l`` để xem có thể dùng những lệnh nào
- ![img](img/sudo.png?raw=true)
- Thử chạy python với file bak.py xem sao
- ![img](img/python.png?raw=true)
- Không ra gì cả. Vậy thử xem file bak.py xem
- ![img](img/bak.png?raw=true)
- Vậy chúng ta sẽ thay thế ``python -c import pty;ptr.spawn("/bin/bash")`` vào file bak.py
- ![img](img/echo.png?raw=true)
- Không có quyền truy cập. Tôi nghĩ là tôi sẽ thử xóa file này và tạo một file mới xem sao
- ![img](img/run.png?raw=true)
- Sau khi xóa và tạo file mới chúng ta đã thành công. Sau đó chạy lại  
``sudo /usr/bin/python3 /home/meliodas/bak.py``
- Chúng ta đã leo được lên quyền root và giờ chỉ cần đọc file root.txt
- ![img](img/root.png?raw=true)
