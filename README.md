# Web Security Lab

基于 Docker 搭建的 Web 安全实验环境。

本项目使用 DVWA（Damn Vulnerable Web Application）复现 SQL Injection 等常见 Web 漏洞，并结合 Apache 日志、Dozzle 容器日志监控以及 ModSecurity WAF，实现对攻击行为的观察、分析与拦截实验。

---

## 1. 项目结构

```text
Browser
↓
ModSecurity WAF (9198)
↓
DVWA Apache (9090)
↓
Apache access.log
↓
Docker logs
↓
Dozzle 实时日志监控
```

---

## 2. 实验环境

| 组件 | 内容 |
|---|---|
| 操作系统 | Windows 11 |
| 容器平台 | Docker Desktop |
| Web 靶场 | DVWA |
| Web 服务 | Apache |
| 日志监控 | Dozzle |
| WAF | ModSecurity + OWASP CRS |
| DVWA 端口 | 9090 |
| WAF 代理端口 | 9198 |

---

## 3. Docker 部署 DVWA

### 3.1 拉取镜像

```bash
docker pull vulnerables/web-dvwa
```

### 3.2 启动 DVWA

```bash
docker run -d --name dvwa -p 9090:80 vulnerables/web-dvwa
```

### 3.3 访问地址

```text
http://localhost:9090
```

### 3.4 默认账号密码

```text
admin / password
```

### 3.5 页面截图

![DVWA登录页](<img width="1001" height="471" alt="屏幕截图 2026-05-24 161846" src="https://github.com/user-attachments/assets/b83bbd10-8cd4-44c9-ae5a-b46754733a0c" />
)

![DVWA主页](<img width="898" height="882" alt="屏幕截图 2026-05-24 162003" src="https://github.com/user-attachments/assets/852cf67c-4ef8-4cb6-8637-d11a3c98726e" />
)

---

## 4. Dozzle 日志监控

### 4.1 启动 Dozzle

```bash
docker run -d --name dozzle -p 9999:8080 amir20/dozzle
```

### 4.2 访问地址

```text
http://localhost:9999
```

### 4.3 作用说明

Dozzle 用于实时查看 Docker 容器日志。

在本实验中，主要用于观察：

- Apache access.log
- HTTP 请求
- SQL Injection payload
- 攻击行为记录

### 4.4 日志截图

![Dozzle容器列表](./images/dozzle-containers.png)

![DVWA日志](./images/dozzle-dvwa-log.png)

---

## 5. SQL Injection 漏洞复现

DVWA Security Level 设置为：

```text
Low
```

进入页面：

```text
DVWA → SQL Injection
```

---

### 5.1 正常请求

输入：

```text
id=1
```

页面正常返回用户信息。

#### 页面截图

![SQLi正常回显](./images/sqli-normal.png)

#### 日志记录

Dozzle 中可看到对应请求：

```text
GET /vulnerabilities/sqli/?id=1&Submit=Submit HTTP/1.1
```

#### 日志截图

![SQLi正常日志](./images/sqli-normal-log.png)

这是一条正常请求。

---

### 5.2 SQL 注入测试

输入 payload：

```sql
' or '1'='1
```

由于 DVWA low 难度没有对用户输入进行过滤，参数被直接拼接进入 SQL 查询，导致 SQL 逻辑被修改，最终成功获取全部数据。

#### 页面截图

![SQLi注入成功](./images/sqli-success.png)

#### 日志记录

Dozzle 中可看到对应请求：

```text
GET /vulnerabilities/sqli/?id=+%27+or+%271%27%3D%271&Submit=Submit HTTP/1.1
```

#### 日志截图

![SQLi注入日志](./images/sqli-success-log.png)

其中：

```text
%27
```

是单引号 `'` 的 URL 编码，可以明显看到 `or` 等 SQL Injection 特征。

---

## 6. Apache 日志分析

通过 Dozzle 可以直接观察 Apache access.log。

日志字段中常见信息包括：

- 客户端 IP
- 请求路径
- 请求参数
- HTTP 状态码
- 来源页面
- User-Agent

### 示例

```text
172.17.0.1 - - [21/May/2026:09:40:09 +0000] "GET /vulnerabilities/sqli/?id=+%27+or+%271%27%3D%271&Submit=Submit HTTP/1.1" 200 1853 "http://localhost:9090/vulnerabilities/sqli/?id=%27+or+%271%27%3D%271&Submit=Submit" "Mozilla/5.0 ..."
```

### 说明

- `172.17.0.1` 是 Docker bridge 网段地址
- `200` 表示请求成功
- 请求参数中包含 SQL Injection payload
- 攻击行为可在日志中完整还原

---

## 7. ModSecurity WAF 接入

为了模拟真实 Web 防护环境，本项目引入：

- ModSecurity
- OWASP CRS（Core Rule Set）

形成如下结构：

```text
Browser
↓
ModSecurity WAF
↓
DVWA
```

---

### 7.1 WAF Reverse Proxy 配置

ModSecurity 容器通过 Nginx reverse proxy 转发流量：

```nginx
proxy_pass http://host.docker.internal:9090/;
```

浏览器访问：

```text
http://localhost:9198
```

请求会先经过：

```text
ModSecurity → OWASP CRS 规则检测
```

然后再转发到 DVWA。

### 7.2 WAF 页面截图

![WAF入口页](./images/waf-home.png)

![WAF配置](./images/waf-config.png)

---

## 8. WAF 攻击检测实验

再次发送 SQL Injection payload：

```sql
' or '1'='1
```

请求会先进入：

```text
ModSecurity WAF
```

WAF 会根据 OWASP CRS 规则进行检测。

### 8.1 可能出现的结果

- 请求被拦截，返回 `403 Forbidden`
- 请求被放行，继续进入 DVWA
- 日志中出现规则命中记录

### 8.2 结果截图

![WAF拦截结果](./images/waf-block.png)

![WAF拦截日志](./images/waf-block-log.png)

---

## 9. 日志链路分析

本实验中存在两条访问链路。

### 9.1 原始漏洞环境

```text
Browser
↓
DVWA (9090)
```

直接访问漏洞环境，日志主要记录在 `dvwa` 容器中。

---

### 9.2 WAF 防护环境

```text
Browser
↓
ModSecurity (9198)
↓
DVWA (9090)
```

请求先经过 WAF，日志会同时出现在：

- `blissful_shaw`
- `dvwa`

两个容器中。

---

## 10. 学习收获

通过本实验，学习了：

- Docker 容器化部署
- DVWA Web 漏洞复现
- SQL Injection 原理
- Apache access.log 分析
- Docker logs 日志观察
- Dozzle 容器日志监控
- ModSecurity WAF 部署
- Nginx reverse proxy
- HTTP 请求分析
- Web 攻击链路分析

---

## 11. 后续计划

后续将继续扩展：

- BurpSuite 抓包分析
- sqlmap 自动化注入
- ModSecurity 规则分析
- PHP 源码审计
- WAF 绕过实验
