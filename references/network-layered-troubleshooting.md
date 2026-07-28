# 网络分层排障 - 从物理层到应用层

OSI 七层模型在排障中的实际应用。
核心原则：**从下往上查，不要跳层。**

## 分层排查总览

```
第7层 应用层    HTTP/DNS/SSH/自定义协议   curl/telnet/应用日志
第4层 传输层    TCP/UDP端口                ss/nc/tcpdump
第3层 网络层    IP/路由                    ping/traceroute/ip route
第2层 数据链路层 MAC/ARP/VLAN              arp/ip neigh/ethtool
第1层 物理层    网卡/网线/光模块           ip link/ethtool/mii-tool
```

## 第1层：物理层

```bash
# 网卡状态
ip link show
# eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> -> UP = 正常
# eth0: <BROADCAST,MULTICAST> -> 没有UP = 网卡down了
# LOWER_UP = 物理链路在

# 链路详情
ethtool eth0
# Speed: 1000Mb/s
# Link detected: yes    <- yes=物理连接正常，no=网线/光模块问题

# 修复
ip link set eth0 up

# 云服务器注意：
# 云上没有物理网线概念，但网卡状态仍需检查
# 如果 ethtool 不可用，ip link 的 LOWER_UP 标志就够用
```

**常见问题：**
- 网卡 down（`ip link set eth0 up`）
- 物理机：网线松动/光模块脏/交换机端口故障
- VM：虚拟网卡被 hypervisor 断开

## 第2层：数据链路层

```bash
# ARP 表
ip neigh show
# 10.0.0.1 dev eth0 lladdr 00:11:22:33:44:55 REACHABLE
# 10.0.0.2 dev eth0 lladdr 00:11:22:33:44:66 STALE     <- 缓存过期
# 10.0.0.3 dev eth0 FAILED                              <- ARP不通

# ARP 测试
arping -I eth0 -c 3 <目标IP>
# 如果 arping 通但 ping 不通 = 网络层问题
# 如果 arping 也不通 = 数据链路层问题或目标机不响应ARP

# 清除 ARP 缓存
ip neigh flush all
# 或指定
ip neigh del <IP> dev eth0
```

### 双网卡 ARP Flux 问题

**症状：** 双网卡机器，两块网卡配同网段IP，外部访问时ARP响应混乱，包从一个网卡进但MAC是另一个网卡的。

**诊断：**
```bash
# 看两个网卡是否在同网段
ip addr show
# eth0: 10.0.0.12/24
# eth1: 10.0.0.13/24  <- 同网段 = ARP Flux风险

# 看 ARP 响应行为
cat /proc/sys/net/ipv4/conf/all/arp_ignore
cat /proc/sys/net/ipv4/conf/all/arp_announce
# 0 = 默认行为（会Flux）
```

**修复：**
```bash
# 永久
cat >> /etc/sysctl.d/99-arp.conf << 'EOF'
net.ipv4.conf.all.arp_ignore = 1
net.ipv4.conf.all.arp_announce = 2
net.ipv4.conf.default.arp_ignore = 1
net.ipv4.conf.default.arp_announce = 2
EOF
sysctl -p /etc/sysctl.d/99-arp.conf

# 参数含义：
# arp_ignore=1  只响应目标IP是本网卡IP的ARP请求
# arp_announce=2 发送ARP时总是用本网卡自己的IP
```

## 第3层：网络层

```bash
# 1. 基本连通
ping -c10 <目标IP>
# 0% packet loss = 网络层正常
# 有丢包 = 线路质量问题或中间设备过载
# 100% loss = 路由不通或对端防火墙全拦

# 2. MTU 问题排查
ping -c3 -M do -s 1472 <目标IP>
# -M do = 不分片
# -s 1472 = ICMP payload 1472 + 28(IP+ICMP头) = 1500(标准MTU)
# 如果这个通了但大包传输失败 = MTU问题（某段链路MTU<1500）
# 逐级缩小：1472 -> 1400 -> 1300 找到能通过的值

# 3. 路由追踪
traceroute -n <目标IP>
# 看在哪一跳开始 * * *（超时）
# 可能是那台路由器防火墙/策略问题

# 更好的工具（持续追踪+丢包统计）
mtr -n -c 30 <目标IP>
# Loss% 列 = 每一跳的丢包率
# 如果从第3跳开始丢包 = 问题在到第3跳之间的网络

# 4. 路由表
ip route show
ip route get <目标IP>
# 看默认路由和到目标的路径是否正确

# 5. 云环境特殊注意
# 阿里云/腾讯云内网走 VPC 路由表
# 如果 VPC 路由表没配到目标网段 -> 内网不通
# 安全组在网络层生效，可能静默丢包（不返回ICMP）
```

### 路由问题典型案例

```bash
# 症状：能ping通网关但ping不通外网
ping -c3 <网关IP>        # 通
ping -c3 8.8.8.8         # 不通

# 检查默认路由
ip route show | grep default
# 如果没有 default via xxx -> 没有默认路由
# 修复：
ip route add default via <网关IP>

# 症状：两台同网段机器互相ping不通
# 检查1：ARP
ip neigh show | grep <对端IP>
# 检查2：路由（可能是策略路由/多路由表）
ip rule list
ip route show table all | grep <对端IP>
```

## 第4层：传输层

```bash
# 1. 端口监听状态
ss -tlnp | grep <PORT>
# 详见 port-not-listening.md

# 2. 连接状态统计
ss -tan | awk '{print $1}' | sort | uniq -c | sort -rn
#  ESTAB     50   <- 正常
#  TIME-WAIT 200  <- 可能偏高（短连接多）
#  CLOSE-WAIT 5   <- 如果持续增长 = 应用bug
#  LISTEN    10   <- 正常

# 3. TCP 重传统计
nstat -az | grep -i retrans
# TcpRetransSegs = 重传段数
# 如果重传率高 = 网络质量问题

# 4. 连接队列溢出
# 检查溢出计数
nstat -az | grep -i -E "overflow|drop"
# ListenOverflows = 全连接队列溢出次数
# ListenDrops     = 半连接队列丢弃次数
# 非零且持续增长 = somaxconn 太小 或 应用 backlog 太小

# 调整：
sysctl net.core.somaxconn              # 全连接队列（默认128）
sysctl net.ipv4.tcp_max_syn_backlog    # 半连接队列（默认1024）
```

### TCP 状态详解

| 状态 | 含义 | 排障关注点 |
|------|------|-----------|
| LISTEN | 等待连接 | 正常 |
| SYN-SENT | 已发SYN等SYN-ACK | 卡在这 = 对端不通或丢了SYN |
| SYN-RECV | 收到SYN发了SYN-ACK | 卡在这 = 半连接，可能SYN洪水 |
| ESTABLISHED | 连接已建立 | 正常 |
| FIN-WAIT-1 | 主动关闭，发了FIN | 短暂出现正常，堆积=对端不回ACK |
| FIN-WAIT-2 | 主动关闭，FIN已确认 | 堆积=对端不发FIN（应用bug） |
| TIME-WAIT | 被动关闭方等2MSL | 几千个正常，几万=需要调优 |
| CLOSE-WAIT | 收到FIN但没close() | 堆积=应用代码bug（必须改代码） |
| LAST-ACK | 发了FIN等最后ACK | 短暂正常 |
| CLOSING | 双方同时关闭 | 罕见 |

## 第7层：应用层

```bash
# HTTP 测试
curl -v http://<IP>:<PORT>/          # 详细HTTP交互
curl -I http://<IP>:<PORT>/          # 只看header
curl -o /dev/null -s -w "%{http_code} %{time_total}s\n" http://<IP>/

# HTTPS 证书检查
openssl s_client -connect <IP>:443 -servername <域名>
# 看证书链、有效期、是否受信任

# DNS 测试
dig <域名> +short                     # 快速解析
dig <域名> +trace                     # 完整递归过程
dig @<DNS服务器> <域名>                # 指定DNS
nslookup <域名>

# SSH 测试
ssh -vvv user@<IP>                    # 详细SSH调试

# 通用端口测试
nc -zv <IP> <PORT>                   # TCP
nc -zuv <IP> <PORT>                  # UDP
```

## tcpdump 实战

```bash
# 基本语法：tcpdump -选项 -i 网卡 过滤条件
# -nn  不解析IP和端口为域名（更快）
# -A   ASCII输出（看HTTP等文本协议）
# -X   十六进制+ASCII
# -c N 只抓N个包
# -w   写入pcap文件（用wireshark分析）
# -r   读取pcap文件

# 常用过滤
tcpdump -nn -i eth0 host <IP>                    # 指定IP
tcpdump -nn -i eth0 port 80                       # 指定端口
tcpdump -nn -i eth0 host <IP1> and host <IP2>    # 两台机器之间
tcpdump -nn -i eth0 tcp and port 443              # HTTPS流量
tcpdump -nn -i eth0 icmp                          # ping流量
tcpdump -nn -i any port 53                        # DNS（所有网卡）

# 抓包分析HTTP请求
tcpdump -A -nn -i eth0 port 80 -c 50
# 能看到 GET/POST/Host/Header 等HTTP内容

# 保存后分析
tcpdump -nn -i eth0 port 80 -c 100 -w /tmp/http.pcap
tcpdump -r /tmp/http.pcap -A

# 抓三次握手
tcpdump -nn -i eth0 host <IP> and port <PORT> -c 10
# 正常应该看到：SYN -> SYN-ACK -> ACK

# 看是哪一方先发的FIN
tcpdump -nn -i eth0 host <IP> and port <PORT>
# 谁先发 FIN = 谁先主动关闭
```

## 云环境网络排障特殊注意

| 问题 | 症状 | 特殊原因 |
|------|------|---------|
| 安全组没放行 | 外部连不上但本机正常 | 阿里云/腾讯云安全组在网络层静默丢包 |
| VPC路由不通 | 两台云机器内网不通 | VPC路由表没配到目标网段 |
| 弹性公网IP问题 | 外部偶尔连不上 | EIP带宽限流 / EIP被解绑 |
| NAT网关 | 主动外连失败 | NAT网关SNAT规则/带宽限制 |
| 对等连接 | 跨VPC不通 | VPC Peering没配或路由没指向 |
| 云防火墙 | 特定端口不通 | 云防火墙策略（独立于安全组） |

**云环境排查口诀：本机正常+外部不通=先查安全组再查VPC路由。**
