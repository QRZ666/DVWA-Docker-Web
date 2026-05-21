# DVWA-Docker-Web
我的第一个项目，基于docker环境部署的dvwa，并且使用dozzle查看攻击日志
本项目基于Docker部署DVWA靶场环境，用于复现SQL注入、XSS等常见Web漏洞，并结合Apache日志进行攻击行为分析。

## 环境
- 操作系统：Windows 10
- 容器工具：Docker Desktop
- 靶场：DVWA（vulnerables/web-dvwa）
- 运行端口：9090

  cmd启动：docker run -d --name dvwa -p 9090:80 vulnerables/web-dvwa
  访问：http://localhost:9090
  DVWA默认账密：admin / password
  
  ## SQL注入复现
用一个最简单的sql注入演示过程，dvwa的low难度
### 正常请求
输入id=1，回显如下
<img width="382" height="122" alt="屏幕截图 2026-05-21 173105" src="https://github.com/user-attachments/assets/55b1d0ed-a747-4c28-932f-98374948aa2c" />






在dozzle中，也出现了新的日志
<img width="996" height="57" alt="屏幕截图 2026-05-21 173439" src="https://github.com/user-attachments/assets/4ec19987-a9df-4e0c-925b-af0761e74a9c" />



内容：172.17.0.1 - - [21/May/2026:09:33:58 +0000] "GET /vulnerabilities/sqli/?id=12&Submit=Submit HTTP/1.1" 200 1772 "http://localhost:9090/vulnerabilities/sqli/?id=1&Submit=Submit" "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/148.0.0.0 Safari/537.36 Edg/148.0.0.0"
可以清楚看到?id=1，很明显，这是一次正常行为

但是，当我输入了' or '1'='1（基础的SQL注入语句，这里不细说了），效果如下
<img width="394" height="356" alt="屏幕截图 2026-05-21 173636" src="https://github.com/user-attachments/assets/fb204918-ce20-4f6d-8539-9bcd7621efb5" />



因为DVWA的low难度没有设置过滤，所以直接注入成功了！
在日志里，可以看到
<img width="1051" height="58" alt="屏幕截图 2026-05-21 174041" src="https://github.com/user-attachments/assets/472d9725-c805-438d-9df4-352c77eab416" />





内容：172.17.0.1 - - [21/May/2026:09:40:09 +0000] "GET /vulnerabilities/sqli/?id=+%27+or+%271%27%3D%271&Submit=Submit HTTP/1.1" 200 1853 "http://localhost:9090/vulnerabilities/sqli/?id=%27+or+%271%27%3D%271&Submit=Submit" "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/148.0.0.0 Safari/537.36 Edg/148.0.0.0"
?id=%27+or+%271%27%3D%271，这就是我们刚刚注入的SQL语句
很明显，虽然经过了编码，样子已经变了，但是那个or就是sql注入的危险的信号

## 学习点

- Docker容器化部署Web靶场
- Web请求在服务器日志中的记录方式
- SQL注入的本质是SQL逻辑被拼接修改
- 攻击行为可以通过日志还原
在dozzle上部署靶场，一边攻击，一边看日志，在攻防两端的视角，理解漏洞的出现和利用，比无脑打靶场更有收获
