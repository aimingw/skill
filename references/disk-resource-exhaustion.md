# 磁盘/资源耗尽 - 完整排查指南

磁盘满、inode耗尽、fd耗尽、conntrack耗尽是生产环境高频故障。
这些问题的特点是：表现诡异、连锁反应严重、定位需要特定命令。

## 四种资源耗尽对比

| 类型 | 错误信息 | 系统级检查 | 进程级检查 |
|------|---------|-----------|-----------|
| 磁盘空间 | No space left on device | `df -h` | - |
| inode耗尽 | No space left on device (但df -h有空间) | `df -i` | - |
| fd耗尽 | Too many open files / can't accept | `cat /proc/sys/fs/file-nr` | `ls /proc/<PID>/fd \| wc -l` |
| conntrack | nf_conntrack: table full | `cat /proc/sys/net/netfilter/nf_conntrack_count` | - |

## 磁盘空间耗尽

### 快速定位

```bash
# 1. 哪个分区满了
df -h
# Filesystem      Size  Used Avail Use% Mounted on
# /dev/vda1        50G   48G   0G  100% /        <- 满了

# 2. 哪个目录大（逐层钻下去）
du -sh /* 2>/dev/null | sort -rh | head
# 8.0G  /var
# 5.0G  /home
# 3.0G  /usr

du -sh /var/* 2>/dev/null | sort -rh | head
# 6.0G  /var/log
# 1.5G  /var/lib

du -sh /var/log/* 2>/dev/null | sort -rh | head
# 4.0G  /var/log/nginx
# 2.0G  /var/log/messages
```

### 找大文件

```bash
# 全盘找大于1G的文件
find / -type f -size +1G -exec ls -lh {} \; 2>/dev/null

# 某目录下最大的20个文件
find /var/log -type f -exec ls -lh {} + 2>/dev/null | sort -k5 -rh | head -20

# 按修改时间找最近变大的文件
find /var/log -type f -mtime -1 -exec ls -lh {} + 2>/dev/null | sort -k5 -rh | head
```

### 已删除但未释放的文件（经典坑）

**现象：** `df`显示磁盘满了，但 `du` 统计出来的总量远小于使用量。

**原因：** 文件被删除（unlink）了，但有进程还持有该文件的fd（文件描述符），内核不会释放空间直到所有fd关闭。

```bash
# 找到这些文件
lsof +L1
# 或
lsof | grep deleted

# 输出示例：
# COMMAND   PID  USER  FD  TYPE  DEVICE  SIZE/OFF  NODE  NAME
# nginx    1234  root  9w  REG   253,1   10000000  12345 /var/log/nginx/access.log (deleted)
# 这个文件10G，已删除但nginx还开着

# 修复方法1：重启进程（推荐）
systemctl restart nginx

# 修复方法2：清空fd内容（不重启进程）
# 注意：不是所有进程都支持
cat /dev/null > /proc/1234/fd/9
# 或
truncate -s 0 /proc/1234/fd/9
```

### 紧急清理

```bash
# 清空大日志文件（保留文件本身，只清内容）
truncate -s 0 /var/log/nginx/access.log
truncate -s 0 /var/log/messages

# 删除旧的日志压缩包
find /var/log -name "*.gz" -mtime +30 -delete
find /var/log -name "*.1" -mtime +7 -delete

# 清理 journal 日志
journalctl --vacuum-size=500M    # 只保留500M
journalctl --vacuum-time=3d      # 只保留3天

# 清理/tmp
find /tmp -type f -atime +7 -delete

# 清理包管理器缓存
yum clean all          # Rocky/RHEL
apt clean              # Ubuntu

# 清理 Docker
docker system prune -af    # 清理未使用的镜像/容器/网络/缓存
```

### 永久修复

```bash
# 检查 logrotate 配置
logrotate -d /etc/logrotate.d/nginx    # dry-run检查
# 确认 rotate 数量和 size 限制

# 典型 logrotate 配置
cat /etc/logrotate.d/nginx
/var/log/nginx/*.log {
    daily              # 每天轮转
    rotate 7           # 保留7份
    size 100M          # 超过100M也轮转
    compress           # 压缩旧日志
    missingok          # 日志不存在不报错
    notifempty         # 空文件不轮转
    sharedscripts
    postrotate
        nginx -s reopen    # 通知nginx重新打开日志
    endscript
}

# Docker 日志限制
# /etc/docker/daemon.json
{
    "log-driver": "json-file",
    "log-opts": {
        "max-size": "50m",
        "max-file": "3"
    }
}
```

## inode 耗尽

**现象：** `df -h` 显示有空间，但创建文件报 "No space left on device"。

```bash
# 检查inode使用率
df -i
# Filesystem     Inodes IUsed IFree IUse% Mounted on
# /dev/vda1      3200000 3200000 0  100% /     <- inode满了

# 找哪个目录文件数最多
for d in /* ; do
    echo "$(find $d -type f 2>/dev/null | wc -l) $d"
done | sort -rn | head

# 深入找
find /var -type f | wc -l
find /tmp -type f | wc -l

# 常见元凶：
# 1. 大量小文件（session文件、缓存文件、日志分片）
# 2. 邮件队列（/var/spool/postfix/maildrop）
# 3. 容器层文件（/var/lib/docker/overlay2）

# 清理
find /tmp -type f -mtime +7 -delete
find /var/spool/postfix/maildrop -type f -delete
```

## fd（文件描述符）耗尽

**现象：** 服务报 "Too many open files" / "can't accept connection" / "socket: too many open files"。

### 三层限制检查

```bash
# 层1：系统全局限制
cat /proc/sys/fs/file-max        # 系统最大fd数
cat /proc/sys/fs/file-nr
# 输出：已分配  未使用上限  系统上限
# 12345   0    655350

# 层2：用户级限制
ulimit -n                         # 当前shell的fd限制（默认1024）
cat /etc/security/limits.conf | grep -v "^#" | grep nofile

# 层3：进程级限制
cat /proc/<PID>/limits | grep "open files"
# Max open files    65535    65535    files
```

### 调整限制

```bash
# 临时（当前shell）
ulimit -n 65535

# 永久（用户级）
cat >> /etc/security/limits.conf << 'EOF'
*    soft    nofile    65535
*    hard    nofile    65535
EOF
# 需要重新登录生效

# systemd 服务级
systemctl edit <service>
[Service]
LimitNOFILE=65535

# 确认生效
systemctl show <service> | grep LimitNOFILE
cat /proc/<PID>/limits | grep "open files"
```

### 找到泄漏fd的进程

```bash
# 看哪个进程fd最多
for pid in $(ps -eo pid=); do
    count=$(ls /proc/$pid/fd 2>/dev/null | wc -l)
    [ "$count" -gt 100 ] && echo "$count $pid $(cat /proc/$pid/comm 2>/dev/null)"
done | sort -rn | head -10

# 看某进程打开了什么
ls -l /proc/<PID>/fd/ | head -20
lsof -p <PID> | head -20

# 看fd类型统计
lsof -p <PID> | awk '{print $5}' | sort | uniq -c | sort -rn
```

## conntrack（连接跟踪表）耗尽

**现象：** 新TCP连接无法建立，`dmesg` 报 "nf_conntrack: table full, dropping packet"。

```bash
# 检查
cat /proc/sys/net/netfilter/nf_conntrack_count    # 当前连接跟踪数
cat /proc/sys/net/netfilter/nf_conntrack_max      # 最大值

# 比例
echo "scale=2; $(cat /proc/sys/net/netfilter/nf_conntrack_count) / $(cat /proc/sys/net/netfilter/nf_conntrack_max)" | bc
# > 0.8 = 接近耗尽

# 看哪些连接占了conntrack
conntrack -L 2>/dev/null | head
# 或
cat /proc/net/nf_conntrack | head
```

**修复：**

```bash
# 临时：调大限制
echo 524288 > /proc/sys/net/netfilter/nf_conntrack_max
sysctl -w net.netfilter.nf_conntrack_max=524288

# 同时调hash表大小（应为max的1/4到1/8）
echo 131072 > /proc/sys/net/netfilter/nf_conntrack_buckets

# 永久
cat >> /etc/sysctl.d/99-conntrack.conf << 'EOF'
net.netfilter.nf_conntrack_max = 524288
net.netfilter.nf_conntrack_buckets = 131072
# 缩短超时（减少无用条目占用）
net.netfilter.nf_conntrack_tcp_timeout_established = 3600
net.netfilter.nf_conntrack_tcp_timeout_time_wait = 30
EOF
sysctl -p /etc/sysctl.d/99-conntrack.conf
```

**注意：** 如果不用 NAT/状态防火墙，可以完全关闭 conntrack：
```bash
# 卸载模块（需要没有规则依赖）
modprobe -r nf_conntrack
# 但如果 iptables 有 state/conntrack 规则就无法卸载
```

## 磁盘满连锁故障场景

```
磁盘满
├── 数据库无法写 -> 主从中断 -> 复制堆积 -> 应用读到旧数据
├── 日志无法写 -> 某些服务崩溃退出 -> 连锁故障
├── /tmp无法写 -> 会话文件失败 -> 登录失败 -> 管理员无法SSH
│   (SSH需要写/tmp或用户家目录)
├── PID文件无法写 -> 服务无法启动 -> 看似"服务起不来"
├── 证书/密钥无法读取（目录满了导致读取也异常）-> TLS握手失败
└── Docker无法创建新容器层 -> 容器启动失败
```

**预防措施：**

```bash
# 磁盘使用率监控
# Zabbix/Grafana告警阈值：
# 80% = 告警
# 90% = 紧急
# 95% = 立刻处理

# 自动清理脚本示例
#!/bin/bash
THRESHOLD=90
USAGE=$(df / | tail -1 | awk '{print $5}' | tr -d '%')
if [ "$USAGE" -gt "$THRESHOLD" ]; then
    # 紧急清理journal
    journalctl --vacuum-size=200M
    # 清理旧日志
    find /var/log -name "*.gz" -mtime +7 -delete
    # 通知
    echo "Disk usage ${USAGE}% on /, emergency cleanup executed" | wall
fi
```
