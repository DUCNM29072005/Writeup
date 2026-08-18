Đầu tiên chúng ta sẽ khởi động máy ảo và quét các port của IP bằng công cụ nmap ``nmap -sV 10.129.58.167``
![img](img/nmap.png?raw=true)
Ở đây chúng ta sẽ thấy hai port đang hoạt động đó là SSH (port 22) và một web application (port 3000). Chúng ta hãy thử truy cập IP với port 3000 này
![img](img/dashboard.png?raw=true)
Chúng ta có thể sử dụng công cụ whatweb để xem trang web này sử dụng framework gì
![img](img/whatweb.png?raw=true)
Trang web này sử dụng framwork Next.js, sử dụng một extension trên firefox là Wappalyzer để xác định được phiên bản của Next.js

<p align="center">
  <img src="img/wappalyzer.png?raw=true">
</p>
Đây là phiên bản Next.js 15.0.3. Sau khi search trên google tôi tìm ra được đây là CVE-2025-55182(React2Shell) được công bố cho React Server Components
Đây là một lỗ hổng Remote Code Execution (RCE) trước khi xác thực với CVSS 10.0, nghĩa là attacker không cần tài khoản vẫn có thể thực thi mã trên máy chủ nếu ứng dụng dễ bị ảnh hưởng. Lỗi hổng xả ra do quá trình deserialization không an toàn của các payloads từ các yêu cầu HTTP được gửi đến các React Server Function Endpoints cho phép attacker thực hiện RCE. Kẻ tấn công bắt đầu bằng việc tạo một Fake Chunk có cấu trúc giống dữ liệu hợp lệ của React Flight Protocol và chèn thuộc tính then. Trong JavaScript, mọi object có phương thức then() đều được coi là một Thenable, vì vậy trong quá trình deserialize, React sẽ nhầm đối tượng này là một Promise hợp lệ và tự động gọi phương thức then(). Do phương thức then() hoàn toàn do attacker kiểm soát, kẻ tấn công có thể chiếm quyền điều khiển luồng xử lý (Control Flow) và truy cập vào các đối tượng nội bộ của React, đặc biệt là _response. Từ đây, attacker thực hiện Server-Side Prototype Pollution bằng cách ghi đè các thuộc tính như __proto__, làm thay đổi hành vi của các object trong môi trường Node.js. Cuối cùng, thông qua các gadget có sẵn như process.mainModule.require('child_process').execSync(), attacker có thể thực thi lệnh hệ thống tùy ý trên máy chủ, dẫn đến Remote Code Execution (RCE).
Chúng ta có thể sử dụng công cụ metasploit để khai thác lỗ hổng này
![imt](img/metasploit.png?raw=true)
Có thể thấy chúng ta đã vào được reverse shell với user là node. Sau đó tìm được một file là reactor.db một file SQLite3 sau khi kiểm tra bằng lệnh file. Giờ chúng ta sẽ dump file này để kiểm tra dữ liệu trong đó
![img](img/sqlite3.png?raw=true)
Sẽ dump ra được tên user và mật khẩu đã được băm. Giờ hãy thử crack mật khẩu đã băm này bằng một công cụ CrackStation tìm mật khẩu của user
![img](img/crack.png?raw=true)
Vậy là đã tìm ra mật khẩu của user engineer. Sử dụng kết nối user này bằng ssh và đọc file user.txt
<br>
![img](img/user.png?raw=true)

**Privilege Escalation**
Sử dụng lệnh `netstat -tulpn` để hiển thị các port đang mở và các tiến trình đăng lắng nghe cổng đó
![img](img/netstat.png?raw=true)
Thấy cổng 9229 là cổng mặc định của tính năng Node.js Inspector một trình gỡ lỗi hệ thống của Node.js. Chúng ta hãy kiểm tra tiến trình này xem với lệnh `ps aux | grep node`
![img](img/ps.png?raw=true)
Thấy rằng tiến trình này đang chạy với quyền root. Khi biết cổng này đang chạy dưới quyền root, nó mở ra một con đường leo thang đặc quyền cực kỳ nghiêm trọng vì lý do cốt lõi sau:1. Bản chất của Node.js Inspector (Tính năng gỡ lỗi)Khi một nhà phát triển hoặc quản trị viên khởi động một ứng dụng Node.js bằng cờ --inspect (ví dụ: node --inspect app.js), Node.js sẽ mở ra một "cửa hậu" (backdoor) hợp pháp để nhà phát triển kết nối vào sửa lỗi. Cửa sau này cho phép bất kỳ ai kết nối tới nó đều có quyền gửi và thực thi trực tiếp các đoạn mã JavaScript vào ngay bên trong bộ nhớ của ứng dụng đang chạy. Khi kết nối vào cổng 9229 thành công, ta có toàn quyền tương tác với môi trường Node.js của root.Khi bạn gửi một lệnh JavaScript thông qua hàm child_process.exec(), bản thân Node.js (đang là root) sẽ trực tiếp yêu cầu hệ điều hành chạy câu lệnh đó. Bây giờ chúng ta sẽ kết nối vào cổng 9229 này
![img](img/Pri.png?raw=true)
Sau khi kết nối sử dụng đoạn JavaScript với hàm `exec()` để thực thi lệnh hệ điều hành trên máy chủ, qua đó khai thác quyền của tiến trình Node.js và hoàn thành bước leo thang đặc quyền.
<p align="center">
  <img src="img/root.png?raw=true">
</p>
