# 端口不监听 - 完整排查流程

端口不监听是运维排障最高频的问题之一。
本文覆盖从"端口不通"到"找到根因"的完整流程。

## 快速判断流程图

```
客户端连接 IP:PORT 失败
        |
        v
  ss -tlnp | grep <PORT>
   /              \
  端口在              端口不在
  |                  |
  v                  v
问题在网络/防火墙     问题在服务本身
(跳到"网络层排查")    (继续往下)
        |
        v
  systemctl status <service>
   /          \
  active       inactive/failed
  |            |
  v            v
端口绑定了     看日志找原因
127.0.0.1     (跳到"服务层排查")
不是0.0.0.0
  |
  v
改bind地址
```

## 第一步：确认端口状态

```bash
# 看端口有没有在监听
ss -tlnp | grep <PORT>

# 关键看两列：
# Local Address:Port  -> 0.0.0.0:3306 = 所有网卡（外部可连）
#                       127.0.0.1:3306 = 仅本机（外部连不上）
#                       [::]:3306      = IPv6所有网卡

# ss 各列含义：
# Netid  State  Recv-Q  Send-Q  Local Address:Port  Peer Address:Port  Process
# tcp    LISTEN 0       128     0.0.0.0:3306        0.0.0.0:*          users:(("mysqld",pid=1234,fd=23))
```

### 常见端口绑定问题

| 绑定地址 | 含义 | 外部能连吗 |
|---------|------|-----------|
| `0.0.0.0:PORT` | 监听所有网卡的 IPv4 | 能 |
| `127.0.0.1:PORT` | 仅监听本地回环 | 不能 |
| `10.0.0.12:PORT` | 只监听某个网卡 | 只有该网段能连 |
| `[::]:PORT` | 监听所有网卡 IPv6（通常兼容 IPv4） | 通常能 |
| `[::1]:PORT` | 仅监听 IPv6 本地回环 | 不能 |

**经典坑：** MySQL 默认 `bind-address=127.0.0.1`（Ubuntu），外部连不上。

```bash
# Ubuntu MySQL
grep bind-address /etc/mysql/mysql.conf.d/mysqld.cnf
# bind-address = 127.0.0.1  <- 改成 0.0.0.0
```

## 第二步：本机自连测试

```bash
# TCP 端口探测（不需要nc）
timeout 3 bash -c 'echo > /dev/tcp/127.0.0.1/<PORT> && echo OK' 2>&1
# OK = 端口在监听，本机能连
# Connection refused = 端口没监听
# 超时无输出 = 可能被本地防火墙挡

# 用 nc 测试
nc -zv 127.0.0.1 <PORT>
# Connection to 127.0.0.1 port <PORT> [tcp/*] succeeded!
```

## 第三步：网络层排查（端口在但不通）

如果 `ss` 显示端口在监听，但外部连不上：

```bash
# 1. 防火墙检查（本地）
systemctl is-active firewalld
# active -> 检查规则
firewall-cmd --list-all
firewall-cmd --list-ports
# 没放行就加：
firewall-cmd --permanent --add-port=<PORT>/tcp
firewall-cmd --reload

# iptables 方式
iptables -L -n --line-numbers | grep <PORT>

# nftables（Rocky9/RHEL9）
nft list ruleset | grep <PORT>

# 2. SELinux 检查（Rocky/RHEL）
getenforce
# Enforcing -> setenforce 0 测试
# 详见 selinux-silent-block.md

# 3. 云安全组（最容易被忽略）
# 阿里云/腾讯云/AWS 控制台 -> 安全组规则 -> 入方向
# 确认对应端口和协议已放行
# 这是云服务器"端口不通"最高频原因
```

## 第四步：服务层排查（端口根本没监听）

```bash
# 1. 服务状态
systemctl status <service>
# Active: inactive (dead) / failed / activating
# 看 Main PID 和 ExitCode

# 2. 日志
journalctl -u <service> --since "30 min ago" --no-pager

# 3. 配置检查（各服务语法检查命令）
nginx -t                           # Nginx
named-checkconf                    # BIND DNS
named-checkzone <domain> <file>    # DNS zone file
httpd -t                            # Apache
mysqld --validate-config           # MySQL 8.0+
redis-server /etc/redis/redis.conf --check-system  # Redis（部分版本）
sshd -t                             # SSH
postfix check                       # Postfix

# 4. 直接运行二进制（绕过systemd）
# systemd 会吃掉一些错误信息，直接运行能看到真实报错
/usr/sbin/nginx -g "daemon off;"
/usr/sbin/named -g -c /etc/named.conf
# 前台运行，看报错，Ctrl+C 退出
```

## 常见端口不监听原因清单

| 原因 | 症状 | 验证方法 |
|------|------|---------|
| 服务没启动 | systemctl inactive | `systemctl status` |
| 配置文件语法错 | 日志有语法错误 | 语法检查命令 |
| 端口被占用 | Address already in use | `ss -tlnp \| grep <PORT>` |
| 绑定127.0.0.1 | 本机能连外部不行 | `ss -tlnp` 看 Local Address |
| SELinux拦截 | 无日志端口不出现 | `setenforce 0` 测试 |
| 防火墙没放行 | 本机能连外部不行 | `firewall-cmd --list-all` |
| 云安全组没放行 | 本机能连外部不行 | 控制台检查 |
| 依赖服务没起 | 日志报连接失败 | 检查数据库/缓存状态 |
| 权限不够 | Permission denied | `ls -l` 检查文件/目录权限 |
| 配置路径不对 | 找不到配置文件 | `systemctl cat <service>` 看 ExecStart |
| systemd Type=notify 卡住 | activating 超时 | 改 Type=forking 或检查notify逻辑 |
| 端口号超范围 | 端口 > 65535 | 检查配置文件 |
| 内核参数限制 | 端口在范围内但绑定失败 | `sysctl net.ipv4.ip_local_port_range` |

## 跨机端口排查模板

从客户端到服务端逐层检查：

```bash
# === 在客户端机器上 ===

# 1. ping 服务端IP（网络层通不通）
ping -c3 <SERVER_IP>

# 2. traceroute（看路径哪一跳开始不通）
traceroute -n <SERVER_IP>

# 3. TCP端口探测
nc -zv <SERVER_IP> <PORT>
# 或
timeout 3 bash -c 'echo > /dev/tcp/<SERVER_IP>/<PORT> && echo OK'

# === 在服务端机器上 ===

# 4. 确认端口在监听
ss -tlnp | grep <PORT>

# 5. 本机自连
timeout 3 bash -c 'echo > /dev/tcp/127.0.0.1/<PORT> && echo OK'

# 6. 防火墙
firewall-cmd --list-ports
iptables -L -n | grep <PORT>

# 7. SELinux
getenforce

# 8. 抓包看请求有没有到
tcpdump -nn -i any port <PORT> -c 10
# 然后在客户端发起连接
# 如果抓到 SYN 但没有 SYN-ACK = 服务/防火墙问题
# 如果连 SYN 都没有 = 网络路由问题
```
