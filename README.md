# DVWA-Docker-Web
1. 项目简介

本项目基于 Docker 构建 DVWA（Damn Vulnerable Web Application）漏洞实验环境，通过 Web 攻击复现（SQL Injection）结合 Apache 访问日志与 Dozzle 容器日志监控，实现对攻击行为的可观测分析。
2. 环境信息

操作系统：Windows 10
容器平台：Docker Desktop
靶场环境：DVWA（vulnerables/web-dvwa）
日志工具：Dozzle
访问端口：9090

cmd启动：docker run -d --name dvwa -p 9090:80 vulnerables/web-dvwa
访问：http://localhost:9090
DVWA默认账密：admin / password
  
3.SQL注入复现
用一个最简单的sql注入演示过程，dvwa的low难度

先输入id=1正常请求，回显如下
<img width="382" height="122" alt="屏幕截图 2026-05-21 173105" src="https://github.com/user-attachments/assets/55b1d0ed-a747-4c28-932f-98374948aa2c" />






在dozzle中，也出现了新的日志
<img width="996" height="57" alt="屏幕截图 2026-05-21 173439" src="https://github.com/user-attachments/assets/4ec19987-a9df-4e0c-925b-af0761e74a9c" />



内容：172.17.0.1 - - [21/May/2026:09:33:58 +0000] "GET /vulnerabilities/sqli/?id=12&Submit=Submit HTTP/1.1" 200 1772 "http://localhost:9090/vulnerabilities/sqli/?id=1&Submit=Submit" "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/148.0.0.0 Safari/537.36 Edg/148.0.0.0"




可以清楚看到?id=1，很明显，这是一次正常行为

但是，当我输入了' or '1'='1（基础的SQL注入语句，这里不细说了），效果如下
<img width="394" height="356" alt="屏幕截图 2026-05-21 173636" src="https://github.com/user-attachments/assets/fb204918-ce20-4f6d-8539-9bcd7621efb5" />



因为DVWA的low难度没有设置过滤，参数进入SQL语句，导致SQL结构被拼接。所以直接注入成功了！
在日志里，可以看到
<img width="1051" height="58" alt="屏幕截图 2026-05-21 174041" src="https://github.com/user-attachments/assets/472d9725-c805-438d-9df4-352c77eab416" />





内容：172.17.0.1 - - [21/May/2026:09:40:09 +0000] "GET /vulnerabilities/sqli/?id=+%27+or+%271%27%3D%271&Submit=Submit HTTP/1.1" 200 1853 "http://localhost:9090/vulnerabilities/sqli/?id=%27+or+%271%27%3D%271&Submit=Submit" "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/148.0.0.0 Safari/537.36 Edg/148.0.0.0"
?id=%27+or+%271%27%3D%271



通过 Dozzle 观察 Docker 容器日志，可实时获取 Apache access.log 内容。
这就是一条明显的SQL注入！
日志字段说明：

客户端 IP：172.17.0.1（Docker bridge）
请求路径：/vulnerabilities/sqli/
请求参数：包含 SQL injection payload
HTTP 状态码：200


4.攻击链还原
浏览器输入 → HTTP请求 → Apache解析 → access.log记录 → Docker日志输出 → Dozzle展示


5. 安全分析结论
SQL 注入发生在参数拼接阶段
用户输入未经过过滤直接进入 SQL 查询
攻击请求可以在 HTTP 日志中完整还原
容器日志（Docker logs）可用于实时监控攻击行为

6. 项目收获
掌握 Docker 靶场部署流程
理解 SQL 注入攻击原理
学习 HTTP 请求在服务器日志中的表现形式
建立“攻击行为 → 日志记录 → 行为还原”的分析思路
