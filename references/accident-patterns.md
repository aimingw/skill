# 生产高频事故模式库

生产环境最高频的故障模式，遇到时直接套经验。

## 模式1：OOM Killer 杀进程

**典型场景：** 某个Java/Python/数据库进程突然消失，服务挂了，手动重启后又正常。

**排查：**
```bash
# 1. 确认是OOM
dmesg -T | grep -i "killed process"
# [Mon Jan 1 10:00:00 2024] Killed process 12345 (java) total-vm:8G, anon-rss:6G

# 2. 看内存历史趋势
sar -r    # 如果装了sysstat

# 3. 查进程的内存限制
cat /proc/<PID>/cgroup     # 是否在容器里有限制
systemctl show <service> | grep -i memory  # systemd MemoryLimit
```

**修复：**
- 临时：加swap、限制进程内存、调整oom_score_adj保护关键进程
- 永久：找到内存泄漏根因、扩容、调整JVM堆/Redis maxmemory

**保护关键进程：**
```bash
echo -1000 > /proc/<PID>/oom_score_adj    # 禁止被杀
```

## 模式2：磁盘满连锁故障

**典型场景：** 磁盘满了导致数据库不能写、日志不能记、服务各种报错，一恢复磁盘全部正常。

**连锁反应路径：**
```
磁盘满 -> 数据库无法写binlog -> 主从中断 -> 应用报错 -> 告警风暴
磁盘满 -> 日志无法写 -> 某些服务直接退出 -> 连锁故障
磁盘满 -> /tmp不能写 -> 会话文件失败 -> 登录异常
```

**排查：**
```bash
df -h                              # 哪个分区满了
du -sh /* 2>/dev/null | sort -rh | head  # 哪个目录大
lsof +L1                           # 有没有已删未释放的文件
```

**修复：**
- 紧急：`truncate -s 0 /var/log/xxx.log`
- 找根因：logrotate没配/数据库慢查询日志狂涨/容器日志没限制
- 预防：配磁盘使用率告警（80%告警，90%紧急）

## 模式3：连接数耗尽

**两种类型：**

**类型A - fd耗尽（单进程级）：**
```
错误：Too many open files
排查：ls /proc/<PID>/fd | wc -l  对比  cat /proc/<PID>/limits | grep "open files"
修复：提高 ulimit -n / systemd LimitNOFILE / /etc/security/limits.conf
```

**类型B - conntrack耗尽（系统级）：**
```
错误：nf_conntrack: table full, dropping packet
排查：cat /proc/sys/net/netfilter/nf_conntrack_count 对比 nf_conntrack_max
修复：临时提高 nf_conntrack_max；永久调整sysctl
症状：ping通但TCP建连失败，dmesg里有conntrack满的日志
```

## 模式4：TIME_WAIT 堆积

**典型场景：** Nginx/应用突然连接不上后端，ss看到大量TIME_WAIT。

**排查：**
```bash
ss -tn state time-wait | wc -l      # TIME_WAIT数量
sysctl net.ipv4.tcp_max_tw_buckets  # 上限
```

**判断标准：**
- 几千个TIME_WAIT是正常的（TCP正常流程）
- 几万个 + 新连接失败 = 问题
- 常见于短连接高频请求场景（HTTP API没有keepalive）

**修复：**
```bash
# 临时
sysctl -w net.ipv4.tcp_tw_reuse=1
# 永久写入 /etc/sysctl.d/
# 根本：应用层改用长连接/连接池
```

## 模式5：CLOSE_WAIT 堆积（应用bug）

**典型场景：** 某服务fd持续增长，最终fd耗尽。

**排查：**
```bash
ss -tn state close-wait | wc -l     # CLOSE_WAIT数量
```

**判断：**
- CLOSE_WAIT = 对端发了FIN但本地应用没调close()
- 堆积 = 应用代码忘记关闭连接（常见于没有finally块关闭资源）
- 不是调内核参数能解决的，必须改代码

## 模式6：DNS解析失败导致的诡异问题

**典型场景：** 服务间歇性报错，重启就好，过一会儿又报错。

**排查：**
```bash
# 1. 看 resolv.conf
cat /etc/resolv.conf
# nameserver 指向的DNS挂了/超时了？

# 2. 测试DNS
nslookup <域名>
dig <域名> +time=2 +tries=1    # 带超时测试
# 如果解析慢(>1秒)，DNS是嫌疑

# 3. 看是否缓存了错误结果
nscd -g 2>/dev/null            # nscd缓存
resolvectl status              # systemd-resolved

# 4. 对比IP直连 vs 域名
curl http://<IP> -o /dev/null -w "%{time_total}\n"
curl http://<域名> -o /dev/null -w "%{time_total}\n"
# IP快域名慢 = DNS问题
```

**修复：**
- 临时：换DNS服务器（8.8.8.8 / 114.114.114.114）
- 本地缓存：配置nscd或systemd-resolved
- 应用层：减少DNS查询频率/配置DNS缓存/使用IP直连

## 模式7：时钟不同步导致的认证故障

**典型场景：** Kerberos/SSL/mTLS认证间歇失败，但密码/证书明明没过期。

**排查：**
```bash
date                            # 本机时间
ssh <target> date                # 对端时间
chronyc tracking                # NTP同步状态
# System time 和 Last offset 差距大 = 时钟漂移
```

**影响范围：**
- SSL/TLS证书验证（时间不在validity范围内）
- Kerberos票据（时间戳误差必须<5分钟）
- JWT token（exp字段过期）
- 数据库时间字段（主从时间不一致导致数据混乱）
- 分布式锁/租约（TTL计算错误）

**修复：**
```bash
chronyc makestep                # 立即同步
systemctl enable --now chronyd   # 确保开机启动
```

## 模式8：SELinux 静默拦截

**典型场景：** 服务能启动、配置没报错、但端口不监听或连接被拒，日志里干干净净。

**排查：**
```bash
getenforce                       # Enforcing?
setenforce 0                     # 临时关闭
# 关闭后问题消失 -> 100%是SELinux
# 详见 references/selinux-silent-block.md
```

**口诀：服务能启动、端口不监听、日志全干净、检查SELinux。**

## 快速匹配表

| 关键词/错误信息 | 最可能的原因 | 第一步检查 |
|----------------|-------------|-----------|
| `Connection refused` | 端口没监听/服务没起 | `ss -tlnp \| grep <PORT>` |
| `Connection timed out` | 防火墙/安全组/网络不通 | `ping` + `traceroute` |
| `Permission denied` | 权限/SELinux/AppArmor | `ls -l` + `getenforce` |
| `No space left on device` | 磁盘满/inode满 | `df -h` + `df -i` |
| `Too many open files` | fd耗尽 | `ls /proc/<PID>/fd \| wc -l` |
| `Cannot assign requested address` | 端口耗尽/TIME_WAIT堆积 | `ss -s` |
| `Address already in use` | 端口冲突 | `ss -tlnp \| grep <PORT>` |
| `No route to host` | 路由问题/对端防火墙 | `ip route get <IP>` |
| `killed process` (dmesg) | OOM Killer | `dmesg \| grep killed` |
| `segmentation fault` | 程序bug/依赖库版本不对 | `dmesg \| grep segfault` |
| `nginx: [emerg] bind() failed (98)` | 80端口被占/旧进程未退出 | `ss -tlnp \| grep :80` |
| `502 Bad Gateway` | 后端挂了/SELinux拦截 | `curl backend:port` + `getsebool` |
| `504 Gateway Timeout` | 后端响应太慢/不可达 | 检查后端负载/网络 |
| `java.net.UnknownHostException` | DNS解析失败 | `nslookup <域名>` |
| `SSLError: certificate verify failed` | 证书过期/时钟不同步 | `openssl s_client` + `date` |
| `mysql: Access denied for user` | 密码错/主机不匹配/认证插件 | `SELECT user,host,plugin FROM mysql.user` |
| `docker: Cannot connect to daemon` | Docker没起/socket权限 | `systemctl status docker` |
