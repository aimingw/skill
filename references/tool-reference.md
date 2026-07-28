# 排障工具速查

按场景分类，每个工具列出最常用的参数组合和关键指标。

## CPU 分析

| 工具 | 用途 | 常用命令 | 关键指标 |
|------|------|---------|---------|
| top | 快速看全局 | `top -bn1 \| head -20` | load average、%us/%sy/%wa |
| htop | 交互式top | `htop` | F5看进程树 |
| vmstat | 系统整体状态 | `vmstat 1 5` | r列(运行队列)、cs(上下文切换)、wa(IO等待) |
| pidstat | 进程级CPU | `pidstat -u -p <PID> 1` | %CPU |
| mpstat | 多核CPU | `mpstat -P ALL 1` | 各核使用率是否不均 |
| perf | 内核级分析 | `perf top` / `perf record` | 热点函数调用栈 |

**load average 解读：**
- 三个数字 = 1分钟/5分钟/15分钟的平均运行队列长度
- 单核CPU：load > 1 = 过载；32核CPU：load > 32 = 过载
- load高但CPU不高 = IO等待型瓶颈（看vmstat的wa列和b列）

**%Cpu(s) 各列含义：**
- us: 用户态CPU（应用计算） / sy: 内核态（系统调用/中断/锁竞争）
- wa: IO等待 / hi: 硬中断 / si: 软中断 / st: 虚拟化被偷

**vmstat 关键列：**
- r: 运行队列 > CPU核数 = CPU不够
- b: D状态进程 > 0且持续 = IO瓶颈
- si/so: swap换入换出 > 0 = 内存不足
- cs: 上下文切换 > 50000/s = 可能锁竞争/线程太多

## 内存分析

| 工具 | 用途 | 常用命令 | 关键指标 |
|------|------|---------|---------|
| free | 内存概况 | `free -h` | available(真正可用) |
| top | 进程内存 | `top -o %MEM` | RES(物理内存)、VIRT(虚拟内存) |
| ps | 进程排序 | `ps aux --sort=-%mem \| head` | %MEM排序 |
| smem | PSS/RSS/USS | `smem -rs pss \| head -20` | PSS比RES更准确 |
| pmap | 进程内存映射 | `pmap -x <PID>` | 各内存段大小 |
| vmstat | swap活动 | `vmstat 1 5` | si/so > 0 = 内存不足 |

**free 命令核心原则：看 available 不看 free。**
- free少不代表内存不足（可能被cache用了，需要时自动释放）
- available少才是真的内存不足

**OOM Killer 机制：**
- 内核在内存极度不足时自动杀进程释放内存
- 选择依据：oom_score（基于内存占用+运行时间+优先级）
- `cat /proc/<PID>/oom_score` 查看分数
- `echo -1000 > /proc/<PID>/oom_score_adj` 保护关键进程

## 磁盘 IO 分析

| 工具 | 用途 | 常用命令 | 关键指标 |
|------|------|---------|---------|
| df | 磁盘空间 | `df -h` / `df -i` | 空间/inode使用率 |
| du | 目录大小 | `du -sh /path/* \| sort -rh` | 哪个目录占空间 |
| iostat | IO性能 | `iostat -x 1 5` | %util、await、svctm |
| iotop | 谁在读写 | `iotop -oP` | 进程级IO读写速率 |
| pidstat | 进程级IO | `pidstat -d -p <PID> 1` | 读/写KB/s |
| lsof | 已删文件 | `lsof +L1` | 已删除但fd未关的大文件 |

**iostat 关键指标：**
- %util: 接近100% = IO瓶颈
- await: IO响应时间，>10ms算慢，>100ms很慢
- 高%util + 低await = 请求量大但设备扛得住（正常繁忙）
- 高%util + 高await = 设备确实扛不住了

**判断模式：**

| %util | await | 含义 |
|-------|-------|------|
| 低 | 低 | 设备空闲 |
| 高 | 低 | 请求量大但扛得住 |
| 高 | 高 | 设备扛不住了 |
| 低 | 高 | 设备有问题（坏道/驱动） |

## 网络分析

| 工具 | 用途 | 常用命令 |
|------|------|---------|
| ss | socket状态 | `ss -tlnp`(监听) / `ss -tnp`(连接) / `ss -s`(汇总) |
| tcpdump | 抓包 | `tcpdump -nn -i eth0 port 80 -c 50 -w file.pcap` |
| nc | 端口探测 | `nc -zv <IP> <PORT>`(TCP) / `nc -zuv <IP> <PORT>`(UDP) |
| mtr | 路由+丢包 | `mtr -n -c 30 <IP>` |
| ping | 连通性 | `ping -c10 -i0.1 <IP>` / `ping -c10 -s 1472 -M do <IP>`(MTU) |
| ip | 网络配置 | `ip addr` / `ip route` / `ip neigh` / `ip link` |
| nstat | 内核网络统计 | `nstat -az \| grep -i retrans` |
| conntrack | 连接跟踪 | `cat /proc/sys/net/netfilter/nf_conntrack_count` |

**ss State 列解读：**
- LISTEN(监听) / ESTAB(已连接) / TIME-WAIT(等待关闭) / CLOSE-WAIT(对端关闭本地没关) / SYN-RECV(收到SYN等待ACK)
- Recv-Q/Send-Q: 非零 = 处理速度跟不上

**TCP 状态排障关注点：**

| 状态 | 正常 | 异常堆积含义 |
|------|------|-------------|
| TIME-WAIT | 几千个 | 几万+新连接失败 = 需调tw_reuse |
| CLOSE-WAIT | 几个 | 持续增长 = 应用bug（没调close()），必须改代码 |
| SYN-RECV | 少量 | 堆积 = SYN洪水攻击 |
| FIN-WAIT-2 | 短暂 | 堆积 = 对端不发FIN（应用bug） |

## 进程分析

| 工具 | 用途 | 常用命令 |
|------|------|---------|
| ps | 进程列表 | `ps aux` / `ps -ef` / `ps aux --sort=-%cpu \| head` |
| pstree | 进程树 | `pstree -p <PID>` |
| pgrep | 找进程 | `pgrep -f <关键词>` / `pgrep -a nginx` |
| lsof | 进程打开的文件 | `lsof -p <PID>` / `lsof -i :80` |
| strace | 系统调用追踪 | `strace -p <PID> -c`(统计) / `strace -p <PID> -e trace=network` |
| pstack | 进程栈 | `pstack <PID>`（看进程卡在哪个函数） |
| /proc | 内核视角 | `cat /proc/<PID>/status` / `ls /proc/<PID>/fd \| wc -l` |

## 日志分析

| 工具 | 用途 | 常用命令 |
|------|------|---------|
| journalctl | systemd日志 | `journalctl -u <svc> --since "1h ago" --no-pager -f` |
| grep | 日志搜索 | `grep -i "error" /var/log/messages` |
| tail | 实时日志 | `tail -f /var/log/xxx` / `tail -100f /var/log/xxx` |
| zgrep | 压缩日志 | `zgrep "error" /var/log/messages-*.gz` |
| awk | 日志统计 | `awk '{print $1}' access.log \| sort \| uniq -c \| sort -rn \| head` |
| dmesg | 内核日志 | `dmesg -T \| tail -50` |
| last | 登录记录 | `last -20` / `lastb -20`(失败登录) |

**journalctl 常用组合：**
```bash
journalctl -u <svc> --since "30 min ago" --no-pager     # 按服务
journalctl -p err --since today                          # 按优先级(err及以上)
journalctl --since "10:00" --until "11:00"               # 按时间范围
journalctl -u <svc> -f                                   # 实时跟踪
journalctl -k --since "1 hour ago"                        # 只看内核消息
journalctl --since "1 hour ago" > /tmp/incident.log       # 导出
```

## 系统限制检查

```bash
# 文件描述符
ulimit -n                           # 当前shell的fd限制
cat /proc/<PID>/limits | grep "open files"  # 某进程的fd限制
cat /proc/sys/fs/file-max           # 系统全局fd上限
cat /proc/sys/fs/file-nr            # 已分配/无限制/最大

# 进程数
ulimit -u                           # 用户最大进程数
cat /proc/sys/kernel/pid_max        # 系统最大PID数

# 连接跟踪（conntrack）
cat /proc/sys/net/netfilter/nf_conntrack_max       # 最大连接跟踪数
cat /proc/sys/net/netfilter/nf_conntrack_count     # 当前连接跟踪数

# TCP 相关
sysctl net.ipv4.tcp_max_tw_buckets   # TIME_WAIT上限
sysctl net.core.somaxconn             # 全连接队列大小
sysctl net.ipv4.tcp_max_syn_backlog   # 半连接队列大小

# 用户资源限制
cat /etc/security/limits.conf | grep -v "^#"
cat /etc/security/limits.d/*.conf
```
