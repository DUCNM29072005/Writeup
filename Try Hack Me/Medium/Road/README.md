- Chúng ta sẽ khởi động máy ảo và quét các port của IP bằng nmap:`nmap -sV 10.49.173.255`
![img](img/Screenshot%202026-03-06%20225656.png?raw=true)
- Từ kết quả của nmap có thể thấy 2 port là 22 và 80 đang được mở
- Chúng ta sẽ thử mở IP này trên port 80
![img](img/Screenshot%202026-03-06%20230031.png?raw=true)
- Khi click vào Merchant Central 
![img](img/Screenshot%202026-03-06%20231027.png?raw=true)
- Chúng ta sẽ thấy trang đăng nhập. Thử đăng kí và vào trang chủ của web
- Khi vào được trang chủ. Trong profile sẽ có một chỗ để upload file img, có thể thấy chỉ quyền admin mới có thể upload file
![img](img/Screenshot%202026-03-06%20232302.png?raw=true)
- Thử đổi password trong Reset User dùng burpsuite để chặn và đổi username thành email của admin
![img](img/Screenshot%202026-03-06%20233400.png?raw=true)
- Đã đổi thành công và đăng nhập vào với quyền admin
![img](img/Screenshot%202026-03-06%20233520.png?raw=true)
- Tuy nhiên nếu upload file lên đây thì sẽ chưa thể kích hoạt mã độc. Thử xem source của trang web
![img](img/Screenshot%202026-03-06%20233753.png?raw=true)
- Có thể thấy các file được upload tại `/v2/profileimages/`
- Sau đó upload lên đây một file mã độc php và lắng nghe tại port đó
- Sau đó kích hoạt mã độc tại địa chỉ này và tên của file đã upload
![img](img/Screenshot%202026-03-06%20234108.png?raw=true)
- Sau đó đọc file user.txt trong thư mục của user webdeveloper
![img](img/Screenshot%202026-03-06%20234311.png?raw=true)
- Tiếp theo, ta sẽ leo quyền root để lấy flag của root. Trước tiên tôi chạy thử lệnh `ss -tl` để hiện thị các cổng TCP đang ở trạng thái lắng nghe
![img](img/Screenshot%202026-03-07%20204149.png?raw=true)
- Có thể thấy đang có một port 27017 đang được lắng nghe. Sau ghi search thì tôi đã thấy một điều thú vị
![img](img/Screenshot%202026-03-07%20205004.png?raw=true)
- Đây là port của mongoDB, ta sẽ kết nối đến IP này 
![img](img/Screenshot%202026-03-07%20205236.png?raw=true)
- Sau khi tìm tôi đã tìm được username và password trong backup
![img](img/Screenshot%202026-03-07%20205722.png?raw=true)
- Sau đó ta sẽ chuyển sang user webdeveloper và kiểm tra bằng `sudo -l`
![img](img/Screenshot%202026-03-07%20210034.png?raw=true)
- Có thê quan sát thấy dòng `env_keep += LD_PRELOAD`, tức là thêm thông số này làm thông số mặc định để thiết lập môi trường LD_preload. Sau khi search tôi đã tìm ra cách leo quyền root
![img](img/Screenshot%202026-03-07%20210722.png?raw=true)
- Đầu tiên sẽ tạo một file rootshell.c trong /tmp 
![img](img/Screenshot%202026-03-07%20211528.png?raw=true)
- Sao đó ta sẽ chạy file này với gcc tạo một file rootshell sau đó set lại môi trường LD_PRELOAD thành file này. Chúng ta đã có được quyền root
![img](img/Screenshot%202026-03-07%20211808.png?raw=true)

