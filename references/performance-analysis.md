# 性能分析 - CPU / 内存 / IO 深度排查

当系统"慢"但没完全挂时，需要深入分析性能瓶颈在哪。
本文覆盖三大资源（CPU/内存/IO）的分析方法和常见瓶颈模式。

## 第一反应：30秒快速判断

```bash
# 一条命令看全局
top -bn1 | head -20
```

解读顺序：
1. load average: 三个数字（1/5/15分钟），看趋势
2. %Cpu(s): us(用户态) / sy(内核态) / wa(IO等待) / id(空闲)
3. MiB Mem: total / free / used / avail
4. 进程列表: 按%CPU排序，看谁在吃资源

**快速判断表：**

| 指标 | 正常 | 异常 | 方向 |
|------|------|------|------|
| us% | <70% | >90% | 用户态CPU瓶颈，看具体进程 |
| sy% | <30% | >50% | 内核态瓶颈，可能系统调用/中断/上下文切换太多 |
| wa% | <5% | >20% | IO等待严重，磁盘是瓶颈 |
| id% | >30% | <5% | CPU基本用满 |
| avail Mem | >20% | <10% | 内存不足 |
| load | <CPU核数 | >CPU核数x2 | 过载 |

## CPU 分析

### top 详解

```bash
top -bn1 | head -5
# top - 10:00:00 up 30 days
# Tasks: 150 total, 2 running, 148 sleeping, 0 stopped, 0 zombie
# %Cpu(s): 50.0 us, 10.0 sy, 0.0 ni, 35.0 id, 5.0 wa, 0.0 hi, 0.0 si, 0.0 st
# MiB Mem : 16000.0 total, 3000.0 free, 10000.0 used, 3000.0 buff/cache
```

| 指标 | 含义 | 关注点 |
|------|------|--------|
| us | 用户态CPU | 高=应用计算密集（正常业务/死循环/挖矿） |
| sy | 内核态CPU | 高=系统调用频繁/中断/锁竞争/上下文切换 |
| ni | nice值调整过的CPU | 通常很低 |
| id | 空闲 | 低不代表有问题（业务繁忙正常） |
| wa | IO等待 | 高=磁盘慢，进程在等IO |
| hi | 硬中断 | 高=网卡/磁盘中断过多 |
| si | 软中断 | 高=网络软中断/RCU回调 |
| st | 被偷走（虚拟化） | 高=宿主机超卖，VM抢不到CPU |

### vmstat 详解

```bash
vmstat 1 5
# procs -----------memory---------- ---swap-- -----io---- -system-- ------cpu-----
#  r  b   swpd   free   buff  cache    si   so    bi    bo   in   cs us sy id wa st
#  2  0      0 300000 500000 2500000    0    0    10    20 1000 2000 50 10 35  5  0
#  5  2      0 200000 500000 2500000    0    0   500   100 5000 8000 70 20  5  5  0
```

| 列 | 含义 | 异常值 |
|----|------|--------|
| r | 运行队列长度 | >CPU核数 = CPU不够用 |
| b | D状态(不可中断睡眠)进程数 | >0且持续 = IO瓶颈 |
| swpd | swap使用量 | >0 = 内存曾经不足 |
| si/so | swap换入/换出 | >0 = 当前内存不足 |
| bi/bo | 块设备读/写 | 突然增大 = IO密集 |
| in | 中断数/秒 | 突然增大 = 硬件/网络中断过多 |
| cs | 上下文切换/秒 | >50000 = 可能锁竞争/线程太多 |
| us/sy/wa/id | 同top | 同上 |

**关键判断：**
- r列高 + wa低 = CPU计算瓶颈（加CPU或优化算法）
- b列高 + wa高 = IO瓶颈（优化磁盘/加缓存/SSD）
- cs高 + sy高 = 上下文切换开销（减少线程/改用epoll）
- si/so > 0 = 内存不足在用swap（加内存/限制进程）

### pidstat 进程级分析

```bash
# CPU 使用
pidstat -u 1
# PID    %usr %system  %guest    %CPU   CPU  Command
# 1234   45.0    5.0     0.0    50.0     2   java

# IO 使用
pidstat -d 1
# PID   kB_rd/s   kB_wr/s kB_ccwr/s  Command

# 内存使用
pidstat -r 1
# PID  minflt/s  majflt/s   VSZ    RSS   %MEM  Command

# 上下文切换
pidstat -w 1
# PID      cswch/s nvcswch/s  Command
# 高 cswch/s = 自愿切换多（等IO/锁）
# 高 nvcswch/s = 非自愿切换多（时间片用完被抢占）
```

### perf 火焰图（高级）

```bash
# 安装
yum install perf -y           # Rocky/RHEL
apt install linux-tools-common -y  # Ubuntu

# 采样
perf record -F 99 -p <PID> -g -- sleep 30
# -F 99 = 采样频率99Hz
# -g = 记录调用栈

# 查看
perf report

# 生成火焰图（需要 FlameGraph 工具）
git clone https://github.com/brendangregg/FlameGraph
perf record -F 99 -p <PID> -g -- sleep 30
perf script | FlameGraph/stackcollapse-perf.pl | FlameGraph/flamegraph.pl > /tmp/flame.svg
# 用浏览器打开 svg 文件
```

## 内存分析

### free 命令详解

```bash
free -h
#               total        used        free      shared  buff/cache   available
# Mem:            16Gi       10Gi       1.0Gi       200Mi       5.0Gi       5.5Gi
# Swap:          4.0Gi       0B          4.0Gi
```

| 指标 | 含义 |
|------|------|
| total | 物理内存总量 |
| used | 已使用（不含buffer/cache，新版本） |
| free | 完全未使用 |
| shared | 共享内存（tmpfs等） |
| buff/cache | 内核buffer+cache |
| available | 真正可用（free + 可回收的buffer/cache） |

**核心原则：看 available 不看 free。**
- free少不代表内存不足（被cache用了，需要时会自动释放）
- available少才是真的内存不足

### 进程内存分析

```bash
# 按内存排序
ps aux --sort=-%mem | head -10

# 查看某进程的内存映射
pmap -x <PID> | tail -5
# total kB          RSS      DIRTY
# 能看到各内存段大小

# 查看 /proc 中的内存信息
cat /proc/<PID>/status | grep -E "Vm|Rss"
# VmRSS: 实际使用的物理内存
# VmSize: 虚拟内存大小
# VmPeak: 峰值内存

# 更准确的内存统计（PSS）
smem -rs pss | head -20
# PSS = RSS分摊（共享内存按比例分给各进程）
# 比RSS更准确反映真实内存占用
```

### 内存泄漏判断

```bash
# 方法1：观察RSS是否持续增长
while true; do
    ps -o pid,rss,comm -p <PID>
    sleep 60
done | tee /tmp/mem_trace.log
# 如果RSS持续增长不回落 = 可能泄漏

# 方法2：观察/proc/<PID>/smaps
cat /proc/<PID>/smaps | grep -E "Size|RSS|Private"
# 对比不同时间点，看哪个内存段在增长

# 方法3：观察系统级可用内存趋势
sar -r 1  # 每秒记录一次（需sysstat包）
# kbavail 列持续下降 = 内存可能在被泄漏
```

### OOM Killer 分析

```bash
# 查看历史OOM记录
dmesg -T | grep -i "killed process"
journalctl -k | grep -i "out of memory"

# OOM Killer 杀进程的选择依据
cat /proc/<PID>/oom_score
# 分数越高越容易被杀
# 影响因素：RSS(占大头) + 运行时间(越久越不容易被杀) + nice值

# 保护关键进程不被杀
echo -1000 > /proc/<PID>/oom_score_adj
# 或旧接口
echo -17 > /proc/<PID>/oom_adj

# 完全禁用OOM Killer（不推荐，可能导致系统挂死）
sysctl -w vm.panic_on_oom=1
# 或
echo 1 > /proc/sys/vm/panic_on_oom
```

## 磁盘IO分析

### iostat 详解

```bash
iostat -x 1 5
# Device   r/s     w/s     rkB/s    wkB/s   rrqm/s  wrqm/s  %util  await  svctm
# sda       50.0   100.0   500.0    200.0    5.0     10.0    85.0   5.0    3.0
```

| 指标 | 含义 | 异常值 |
|------|------|--------|
| r/s + w/s | 每秒读写次数 | IOPS指标 |
| rkB/s + wkB/s | 每秒读写量 | 吞吐量指标 |
| %util | 设备繁忙度 | >90% = 接近瓶颈 |
| await | IO响应时间(ms) | >10ms慢，>100ms很慢 |
| svctm | 平均服务时间(ms) | >await说明有排队 |
| rrqm/s + wrqm/s | 合并的读写请求 | 高=请求被合并（正常优化） |

**判断模式：**

| %util | await | 含义 |
|-------|-------|------|
| 低 | 低 | 设备空闲，没问题 |
| 高 | 低 | 请求量大但设备扛得住（正常繁忙） |
| 高 | 高 | 设备确实扛不住了（需要优化/扩容） |
| 低 | 高 | 设备有问题（可能是坏道/驱动问题） |

### iotop 找谁在读写

```bash
# 需要安装
yum install iotop -y    # Rocky
apt install iotop -y    # Ubuntu

# 运行
iotop -oP
# -o 只显示有IO活动的进程
# -P 显示进程而非线程
# TID  PRIO  USER     DISK READ  DISK WRITE  SWAPIN  IO    COMMAND
```

### pidstat 进程级IO

```bash
pidstat -d 1
# PID   kB_rd/s   kB_wr/s kB_ccwr/s  Command
# 1234     0.0    5000.0       0.0    java
# 能看到每个进程每秒读写量
```

## 性能瓶颈判断决策树

```
系统慢
├── CPU瓶颈?
│   ├── top: us% > 80%
│   │   ├── 找到吃CPU的进程: top -o %CPU
│   │   ├── 看是正常业务还是异常: pidstat -u -p <PID>
│   │   ├── 正常业务: 加CPU/优化算法/限流
│   │   └── 异常: strace/perf分析系统调用
│   └── top: sy% > 50%
│       ├── 上下文切换: vmstat看cs列
│       ├── 锁竞争: perf top看锁
│       └── 中断过多: /proc/interrupts
│
├── 内存瓶颈?
│   ├── free -h: available < 10%
│   │   ├── 找内存大户: ps --sort=-%mem
│   │   ├── Java: 检查JVM堆设置
│   │   ├── Redis: 检查maxmemory
│   │   └── 其他: 可能内存泄漏
│   ├── swap正在使用: vmstat看si/so > 0
│   │   └── 立即措施: 限制进程内存/加swap/加物理内存
│   └── OOM记录: dmesg | grep killed
│       └── 保护关键进程: oom_score_adj = -1000
│
├── IO瓶颈?
│   ├── iostat: %util > 90%
│   │   ├── 找IO大户: iotop -oP / pidstat -d
│   │   ├── 大量读: 数据库大查询/全表扫描
│   │   ├── 大量写: 日志/swap/binlog
│   │   └── await高: 磁盘性能不足，换SSD/加缓存
│   └── vmstat: b列 > 0 且持续
│       └── 确认IO等待型瓶颈: wa > 20%
│
└── 网络瓶颈?
    ├── ss -s: 连接数过多
    │   ├── TIME_WAIT堆积: 调tw_reuse/改长连接
    │   ├── CLOSE_WAIT堆积: 应用bug(没close)
    │   └── ESTABLISHED过多: 连接数限制/连接池配置
    ├── 网卡带宽: sar -n DEV / iftop
    │   └── 接近带宽上限: 限流/升级带宽
    └── 重传率高: nstat | grep retrans
        └── 网络质量问题: 检查链路/MTU/网卡参数
```

## 常见性能陷阱

### 1. swap 导致的性能劣化

```bash
# 检查swap状态
free -h | grep Swap
swapon --show

# vmstat看swap活动
vmstat 1 5
# si/so 列 > 0 = 正在使用swap = 严重性能问题
# 即使少量swap使用也会导致间歇性卡顿

# 临时措施
swapoff -a    # 关闭swap（确保内存够用才能做）

# 调整 swappiness（0-100，越低越不爱用swap）
sysctl vm.swappiness
sysctl -w vm.swappiness=10    # 降低swap倾向
# 永久：echo "vm.swappiness=10" >> /etc/sysctl.conf

# 生产环境建议：
# 数据库机器: swappiness=1（几乎不用swap）
# 应用机器: swappiness=10
# 前置机器: swappiness=60（默认）
```

### 2. 上下文切换过高

```bash
# 查看上下文切换
vmstat 1
# cs列 > 50000/秒 = 过高

# 进程级查看
pidstat -w 1
# cswch/s = 自愿切换（等IO/锁 -> 优化IO/减少锁竞争）
# nvcswch/s = 非自愿切换（时间片用完 -> 减少线程数）

# 常见原因：
# - 线程数太多（数百线程争抢CPU）
# - 锁竞争激烈
# - 频繁IO等待
# 解决：减少线程/改用异步IO/减少锁粒度
```

### 3. 中断不均衡

```bash
# 查看中断分布
cat /proc/interrupts | head -5
# 看各CPU核心的中断是否均匀
# 如果某个核的中断特别多 -> 需要调中断亲和性

# 网卡中断优化
cat /proc/irq/<IRQ号>/smp_affinity_list
# 设置中断绑定的CPU核心
echo 0-3 > /proc/irq/<IRQ号>/smp_affinity_list
```
