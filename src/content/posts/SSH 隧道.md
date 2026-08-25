---
title: SSH 隧道与端口转发
pubDatetime: 2026-08-25T22:55:00Z
description: 在远程开发中常用的 SSH 隧道与端口转发。
---

# SSH 隧道与端口转发

## 1. SSH

平时最常见的 SSH 操作 `ssh user@server`

作用是：

```text
本机
  │
  │ SSH 加密连接
  ▼
远程服务器
  │
  └── Shell
```

但 SSH 本质上建立的是一条加密通道：

```text
本机 ──── 加密 SSH 通道 ──── 远程主机
```

这条通道除了传输终端输入输出，还可以传输普通 TCP 网络流量。

因此 SSH 还可以实现：

- 端口转发
- SSH Tunnel / SSH 隧道
- 安全访问远程内部服务

------

## 2. 端口转发

假设 Ubuntu 上有一个服务：

```text
127.0.0.1:8080
```

因为它只监听 Ubuntu 自己的 loopback，显然 Ubuntu 本机可以访问，其他机器不能直接访问。

但可以通过 SSH：

```text
Mac localhost:8080
        │
        ▼
     SSH 隧道
        │
        ▼
Ubuntu 127.0.0.1:8080
```

于是：

```bash
curl http://localhost:8080
```

虽然是在 Mac 上执行，最终访问的却是 Ubuntu 上的服务。

------

## 3. Local Port Forwarding：本地端口转发

这是开发时最常使用的一种 SSH 隧道。

语法：

```bash
ssh -L 本地端口:目标地址:目标端口 user@SSH服务器
```

例如：

```bash
ssh -L 8080:127.0.0.1:8080 user@ubuntu
```

含义是：

```text
Mac localhost:8080
        │
        │ SSH
        ▼
Ubuntu
        │
        ▼
127.0.0.1:8080
```

监听本地 8080，并把进入这个端口的 TCP 流量通过 SSH 转发出去。

------

## 4. `-L` 的三个参数

这个地方非常重要。

```bash
-L 8080:127.0.0.1:8080
```

可以拆成：

```text
-L
│
├── 8080             本机监听端口
│
├── 127.0.0.1        SSH 服务器要访问的目标地址
│
└── 8080             目标端口
```

注意：这里的 `127.0.0.1`指的是 `SSH` 连接的目标的地址，它是站在 **SSH 服务器一侧** 来理解的。

例如 SSH 登录的是 Ubuntu：

```bash
ssh -L 8080:127.0.0.1:8080 user@ubuntu
```

这里的：127.0.0.1:8080 实际上是 Ubuntu 上的 127.0.0.1:8080

------

## 5. 一个容易误解的地方

完整格式：

```bash
ssh -L LOCAL_PORT:TARGET_HOST:TARGET_PORT SSH_HOST
```

存在两类机器：

```text
LOCAL
  Mac 或其他机器

SSH_HOST
  通过 SSH 登录的机器

TARGET_HOST
  SSH 服务器接下来要连接的目标
```

TARGET_HOST 不一定就是 SSH_HOST。

例如：

```text
Mac
 │
 │ SSH
 ▼
跳板机 A
 │
 │ 普通 TCP
 ▼
数据库服务器 B:3306
```

可以：

```bash
ssh -L 3306:10.0.0.20:3306 user@10.0.0.10
```

这里：

```text
10.0.0.10 = SSH 服务器 A
10.0.0.20 = 最终目标服务器 B
```

流量路径：

```text
Mac localhost:3306
        ↓
SSH 到 A
        ↓
A 连接 10.0.0.20:3306
```

所以更准确地说，`-L` 后面的目标地址，是从 SSH 服务器所在网络环境去访问的。

------

## 6. SSH 隧道的作用

例如 Ubuntu 上：

```text
Vite Backend
127.0.0.1:5173

Go Backend
127.0.0.1:8080
```

建立：

```bash
ssh \
  -L 5173:127.0.0.1:5173 \
  -L 8080:127.0.0.1:8080 \
  user@ubuntu
```

Mac 就可以访问 Ubuntu 上的：

```text
http://localhost:5173
http://localhost:8080
```

------

## 7. `ssh -N`

如果只想建立隧道，不需要登录 Shell：

```bash
ssh -N \
  -L 8080:127.0.0.1:8080 \
  user@ubuntu
```

其中：

```text
-N
```

意为不执行远程命令，只建立 SSH 连接和端口转发。适合专门开一个终端维护 SSH 隧道。

------

## 8. 一条 SSH 可以转发多个端口

例如：

```text
Vite        5173
Backend     8080
Prometheus  9090
Grafana     3000
```

可以：

```bash
ssh -N \
  -L 5173:127.0.0.1:5173 \
  -L 8080:127.0.0.1:8080 \
  -L 9090:127.0.0.1:9090 \
  -L 3000:127.0.0.1:3000 \
  user@ubuntu
```

然后 Mac：

```text
localhost:5173
localhost:8080
localhost:9090
localhost:3000
```

SSH 并不是“一条连接只能转发一个端口”。

------

## 9. 写入 `~/.ssh/config`

如果经常使用，不需要每次写很长的命令。

例如 Mac：

```text
~/.ssh/config
```

配置：

```sshconfig
Host dev
    HostName 192.168.1.100
    User myuser

    LocalForward 5173 127.0.0.1:5173
    LocalForward 8080 127.0.0.1:8080
    LocalForward 9090 127.0.0.1:9090
```

以后在执行 ssh 连接 ubuntu 时，就可以自动创建这些 Local Forwad 隧道了。

------

## 10. SSH 隧道传输的是 TCP

SSH Local Forward 本质上做的是：

```text
TCP connection
        ↓
SSH tunnel
        ↓
TCP connection
```

因此它不仅仅能转发 HTTP。

例如还可以转发：

```text
HTTP          8080
HTTPS         443
MySQL         3306
PostgreSQL    5432
Redis         6379
Prometheus    9090
Grafana       3000
SSH           22
```

操作方式与上面同理。

## 11. 本地端口不需要和远程端口相同

例如 Mac 自己已经占用了：

```text
8080
```

完全可以换一个本地端口，例如：

```bash
ssh -L 18080:127.0.0.1:8080 user@ubuntu
```

于是：

```text
Mac localhost:18080
          ↓
Ubuntu localhost:8080
```

访问：` http://localhost:18080 ` 即可。

------

## 12. Local / Remote / Dynamic Forward

SSH 主要有三种转发。

- -L    Local Forward
- -R    Remote Forward
- -D    Dynamic Forward

### `-L` 本地转发

让本地机器可以访问远程网络中的服务。

```text
Local → Remote
```

例如：

```text
Mac → Ubuntu 上的 Vite
Mac → Ubuntu 上的 MySQL
```

### `-R` 远程转发

方向与 -L 想反，让远程机器可以访问本地的服务。

大致：

```text
Remote → Local
```

例如：

```text
云服务器
    │
    ▼
SSH Tunnel
    │
    ▼
你 Mac 上运行的 3000
```

以后遇到 webhook、本地服务临时暴露等场景可能会接触。

### `-D` 动态转发

建立 SOCKS 代理：

```bash
ssh -D 1080 user@server
```

SSH 不再固定转发某一个：`host:port` ，而由 SOCKS 客户端动态决定目的地。

