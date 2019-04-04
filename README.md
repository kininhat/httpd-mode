# httpd-mode
## 1. Khái niệm và cách dùng
### 1.1 Khái niệm
### a. Prefork
Chế độ Prefork không có khái niệm Threads (luồng) -> Việc xử lý request, prefork dùng đúng N tiến trình (tạm gọi tiến trình cha) và tiến trình cha này không tạo ra những tiến trình con để xử lý 1 request tại 1 thời điểm.


### b. Worker
Chế độ Prefork mang có khái niệm Threads (luồng) -> Việc xử lý request, worker dùng đúng N tiến trình (tạm gọi tiến trình cha) và tiến trình cha này sẽ tạo ra những tiến trình con để xử lý 1 request tại 1 thời điểm.

### c. Event
Tương tự Worker, sử dụng thread để phục vụ các request thay vì dùng process, với MPM Event, thread riêng việt sẽ dùng để phục vụ request và giải phóng ngay sau khi xử lý xong, không phụ thuộc vào trạng thái kết nối http.

Prefork: mỗi request gửi đến server sẽ được một process xử lý, điều này gây ra tốn nhiều RAM hơn nếu có nhiều request (do có nhiều process được tạo ra), nhưng ưu điểm là nếu 1 process gặp vấn đề thì không ảnh hưởng đến process khác. Số process được quy định trong file cấu hình apache.conf
Worker: mỗi request gửi đến server sẽ được xử lý bởi một thread (các thread có thể chia sẻ memory với nhau), do đó tiết kiệm RAM hơn, nhược điểm là nếu có một thread gặp vấn đề sẽ ảnh hưởng các thread khác. Có thể quy định số thread trong file cấu hình Apache
Event: tương tự Worker, sử dụng thread để phục vụ các request thay vì dùng process, với MPM Event, thread riêng việt sẽ dùng để phục vụ request và giải phóng ngay sau khi xử lý xong, không phụ thuộc vào trạng thái kết nối http.

### 1.2 Cách dùng
Dựa trên ưa, nhược điểm mà chia cách dùng

|             | Prefork   | Worker      | Event     |
|:-:	        |:-:	      |:-:  	      |:-:        |
| RAM         | Hight  	  |        	    |           |
| CPU         |   	      | Hight  	    |           |
| DISK I/O    | Hight  	  |  Hight 	    | Hight     |
| Secure      |           |             |           |

### 2. PHP Handler 
PHP Handler hiểu theo cách đơn giản là một phương thức định nghĩa cách giao tiếp giữa Webserver và PHP script. Các PHP Handler phổ biến trong Apache gồm: DSO, CGI, FastCGI và suPHP. Các Handler này được xem như 1 module của Apache (nằm trong folder mods-availble)

DSO (Dynamic Shared Object): với handler này, PHP được xem như 1 module của Apache, lúc này giao tiếp giữa Apache và PHP rất nhanh. Tuy sử dụng ít CPU hơn nhưng lại chiếm RAM nhiều hơn (handler này chỉ chạy với MPM Prefork của Apache). Các PHP script được thực thi dưới quyền nobody hoặc Apache, điều này dẫn đến một lỗ hổng bảo mật là các user khác nhau trên hệ thống có thể chạy các php script của nhau. Mặc định Apache sử dụng DSO làm PHP Handler.

CGI: CGI được xem là một hình thức dự phòng khi DSO không có hiệu lực

FastCGI: là một giải pháp có hiệu suất cao thay thế cho CGI. Handler này chạy ít tài nguyên CPU hơn và có tốc độ gần bằng DSO, nhưng tốn nhiều RAM hơn. FastCGI bảo mật hơn do chạy PHP script dưới quyền sở hữu của script đó (nếu thiết lập suExec).
suPHP: PHP script được chạy dưới quyền sở hửu của php script đó. Tất cả PHP script không thuộc 1 user cụ thể nào đó sẽ không thể thực thi được hoặc user này không thể thực hiện được các php script của user khác. Điểm yêu suPHP là sử dụng CPU cao. suPHP được dựa trên nền tảng CGI

### 3. Change mode

[root@apache ~]# httpd -V | grep “Server MPM”
Server MPM:     prefork

Check the prefork configurations used in apache conf file.
<IfModule mpm_prefork_module>
       ServerLimit                               10
       StartServers                              2
       MinSpareServers                    3
       MaxSpareServers                   4
       MaxRequestWorkers            10
       MaxConnectionsPerChild   20
</IfModule>

Understanding directives
ServerLimit:-
Maximum number of child processes that apache is allowed to run.

StartServers:-
Number of child processes created during apache startup.

MinSpareServers:-
Minimum number of idle child processes running. If apache parent process finds that there are idle child processes lesser than minispareservers, it will start the new child processes until it reaches the number setup in MinSpareServers directive.

MaxSpareServers:-
Maximum number of idle child processes allowed to run. If apache parent process finds that there are idle child processes more than the number specified in MaxSpareServer directive, it will terminate them until it reaches the limit configured..

MaxRequestWorkers:-
Maximum number of child processes allowed to run. In case of prefork, this is the number of concurrent requests that apache can handle. Because in prefork configuration, each child process handles one request at a time.

MaxConnectionsPerChild:
If this value is set to 0, a child process will never die.

If this value is set to some value. For example let’s say 10. Then, the child process will die after handling 10 connections. This is recommended to set to some value such as 1000 or more for busy servers to avoid memory leak.

Getting better picture of apache prefork.
As per the above configuration, StartServer is 2 and MinSpareServers is 3. So when apache is started, we should be able to find 3 child processes in few seconds.
[root@apache ~]# ps -C httpd
 PID TTY          TIME CMD
1152 ?        00:00:00 httpd
1154 ?        00:00:00 httpd
1155 ?        00:00:00 httpd
1156 ?        00:00:00 httpd
 

Here, there are 4 processes instead of 3. This is because, the first process is parent process.
To get better picture, run $pstree -alp and scroll through to find the processes.
 ├─httpd,1152 -DFOREGROUND
 │   ├─httpd,1154 -DFOREGROUND
 │   ├─httpd,1155 -DFOREGROUND
 │   └─httpd,1156 -DFOREGROUND
A test is made with 100 concurrent connections. But webserver is handling only 10 requests at a time and website was taking long time to load.
[root@apache ~]# ps -C httpd 
 PID TTY          TIME CMD
1152 ?        00:00:00 httpd
1367 ?        00:00:02 httpd
1661 ?        00:00:01 httpd
1670 ?        00:00:01 httpd
1671 ?        00:00:01 httpd
1680 ?        00:00:01 httpd
1681 ?        00:00:01 httpd
1682 ?        00:00:01 httpd
1688 ?        00:00:00 httpd
1689 ?        00:00:00 httpd
 

[root@apache ~]# ss -tn src :80 or src :443 | wc -l
101
 

Once all the requests are served, the child processes becomes idle. This is when parent process starts killing the children until it reaches MaxSpareServers limit.
In this case, MaxSpareServers is set to 4.
 ├─httpd,1152 -DFOREGROUND
 │   ├─httpd,3750 -DFOREGROUND
 │   ├─httpd,3777 -DFOREGROUND
 │   ├─httpd,3786 -DFOREGROUND
 │   └─httpd,3805 -DFOREGROUND
Lets change ServerLimit to 8 and see what happens to MaxRequestWorkers

[root@apache ~]# ps -C httpd
 PID TTY          TIME CMD
3906 ?        00:00:00 httpd
4078 ?        00:00:05 httpd
4085 ?        00:00:05 httpd
4086 ?        00:00:04 httpd
4095 ?        00:00:04 httpd
4101 ?        00:00:04 httpd
4102 ?        00:00:04 httpd
4111 ?        00:00:03 httpd
4112 ?        00:00:03 httpd
Even though MaxRequestWorkers is 10, only 8 child processes are running. Because it cannot override the value set in ServerLimit.
Optimization:
The memory used by the apache is the total memory used by all the children processes and parent process. To find the total memory used by apache, run the following command.
[root@apache ~]# ps -C httpd -o rss | sed ‘1d’ | awk ‘{x += $1}END{print “Total Memory usage: “ x}’
Total Memory usage: 78284
To find average memory used by each apache process, following command can be used.The memory usage is in kb. It is approximately 78MB here.

[root@apache ~]# ps -C httpd -o rss | sed ‘1d’ | awk ‘{x += $1;y += 1}END{print “Total Memory usage (KB): “ x “\nAverage memory usage (KB): “(x/y)}’
Total Memory usage (KB): 78284
Average memory usage (KB): 15656.8

Memory usage by each apache process is aproximately 16MB here.

You can also run the following command and check the first column.

[root@apache ~]# ps -C httpd -o rss,ucmd
 RSS CMD
14188 httpd
16028 httpd
16024 httpd
16024 httpd
16020 httpd
Let’s allocate 400MB for apache and it should never cross this limit because there could be memory shortage in the server.Let’s say the server is having 1GB ram. 600MB of ram be required for other services such as mysql, exim, php etc.

Each process takes 16M and available ram for apache is 400MB.

Dividing 400 by 16 gives 25.

That means we can run maximum of 25 apache processes. If we exclude parent process, then it leaves us 24 child processes.

So MaxRequestWorkers can be up to 24 or Server limit can be set to 24.

Here it is set according to the memory capacity. But it may not be ideal if the cpu cores are less. During busy hours, the apache could be running maximum number of child processes and processing the requests. It passes the requests to php, mysql to process and provides the result. So php and mysql might consume the resources. So optimise them or try by reducing MaxRequestWorkers to limit the connections.

https://tecadmin.net/apache-prefork-mpm-configuration/
