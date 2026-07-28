# 排障决策树

遇到症状后按顺序执行检查。每棵树对应一类症状。

## 症状1：服务起不来 / systemctl start 失败

```
systemctl start xxx 失败
│
├── 1. 看状态和日志
│   systemctl status xxx          # 看exit code和Active状态
│   journalctl -u xxx --since "10 min ago" --no-pager  # 看详细日志
│
├── 2. 配置文件语法检查
│   nginx -t / named-checkconf / mysql --validate-config / httpd -t
│
├── 3. 直接运行二进制（绕过systemd看真实报错）
│   /usr/sbin/nginx -g "daemon off;"    # 前台运行看报错
│   # 详见 references/systemd-service-fail.md
│
├── 4. 权限检查
│   ls -l <二进制路径>            # 执行权限
│   ls -l <配置文件>             # 读取权限
│   ls -l <数据目录>             # 运行用户是否能写
│   ps -o user= -p <PID>         # 服务以什么用户跑
│
├── 5. 端口冲突
│   ss -tlnp | grep <PORT>       # 端口被谁占了
│   lsof -i :<PORT>
│
├── 6. SELinux（Rocky/RHEL）
│   getenforce                    # Enforcing?
│   setenforce 0                 # 临时关闭测试
│   # 详见 references/selinux-silent-block.md
│
└── 7. 依赖服务
    systemctl is-active <依赖>   # 数据库/缓存/消息队列是否正常
```

## 症状2：端口不通 / 连接拒绝

```
连接 <IP:PORT> 失败
│
├── 1. 端口在不在
│   ss -tlnp | grep <PORT>       # 本机端口监听了吗
│   # 没监听 -> 回到"服务起不来"流程
│
├── 2. 绑定地址对不对
│   ss -tlnp | grep <PORT>       # 127.0.0.1:PORT 还是 0.0.0.0:PORT
│   # 如果绑定127.0.0.1，外部连不上是正常的
│
├── 3. 本机自连
│   timeout 3 bash -c 'echo > /dev/tcp/127.0.0.1/<PORT> && echo OK'
│
├── 4. 远程连通
│   nc -zv <目标IP> <PORT>
│   timeout 3 bash -c 'echo > /dev/tcp/<IP>/<PORT> && echo OK'
│
├── 5. 防火墙
│   systemctl is-active firewalld
│   firewall-cmd --list-all
│   iptables -L -n | grep <PORT>
│
├── 6. SELinux（端口绑定型拦截）
│   getenforce / setenforce 0
│   # 详见 references/selinux-silent-block.md
│
└── 7. 云安全组
    # 阿里云/腾讯云/AWS控制台检查安全组规则
```

## 症状3：响应慢 / 负载高

```
系统响应慢 / top看到load很高
│
├── 1. CPU
│   top -bn1 | head -20          # us高=用户态, sy高=内核态, wa高=IO等待
│   # 常见元凶：挖矿木马、死循环、频繁GC、大量上下文切换
│
├── 2. 内存
│   free -h                      # available还有多少
│   dmesg | grep -i "killed process"   # 检查OOM记录
│   # 常见元凶：Java堆设太大、Redis、内存泄漏
│
├── 3. 磁盘IO
│   iostat -x 1 5                # %util接近100% = IO瓶颈
│   iotop -oP                    # 谁在读写磁盘
│   # 常见元凶：日志狂写、数据库大查询、swap频繁换入换出
│
├── 4. 网络
│   ss -s                        # 连接数统计
│   ss -tn state time-wait | wc -l   # TIME_WAIT堆积
│   ss -tn state close-wait | wc -l  # CLOSE_WAIT堆积（应用bug）
│   # 详见 references/performance-analysis.md
│
├── 5. 进程数/fd
│   ps aux --sort=-%cpu | head
│   ps aux --sort=-%mem | head
│   ls /proc/<PID>/fd | wc -l
│
└── 6. 深度分析
    pidstat -u -p <PID> 1        # 进程级CPU
    pidstat -d -p <PID> 1        # 进程级IO
    strace -c -p <PID>           # 系统调用统计
    # 详见 references/performance-analysis.md
```

## 症状4：磁盘满 / inode耗尽 / fd耗尽

```
No space left on device / 磁盘使用率告警
│
├── 1. 看磁盘空间
│   df -h                        # 哪个分区满了
│   df -i                        # inode是否耗尽（空间有但inode满也报No space）
│
├── 2. 找大文件
│   du -sh /* 2>/dev/null | sort -rh | head   # 逐层钻下去
│   # 常见元凶：/var/log、/tmp、/var/lib/docker、/data
│
├── 3. 已删除但未释放的文件
│   lsof +L1                     # 找已删除但仍被进程持有的文件
│   # 现象：df显示满，但du统计不出来
│   # 修复：重启持有文件的进程，或 kill 对应PID
│
├── 4. 日志文件
│   ls -lhS /var/log/            # 日志按大小排序
│   logrotate -d /etc/logrotate.d/<config>   # dry-run检查
│
├── 5. fd耗尽（不同问题）
│   cat /proc/<PID>/limits | grep "open files"
│   ls /proc/<PID>/fd | wc -l
│   cat /proc/sys/fs/file-nr     # 系统级
│   # 详见 references/disk-resource-exhaustion.md
│
└── 6. 清理
    truncate -s 0 /var/log/xxx.log    # 清空文件内容但保留文件
    # 永久：修logrotate、扩容、配额管理
```

## 症状5：进程异常退出 / OOM / segfault

```
进程没了 / dmesg里看到killed / 应用报segfault
│
├── 1. 是不是被OOM Killer杀了
│   dmesg | grep -i "killed process"
│   dmesg | grep -i "out of memory"
│   # 被杀的进程通常是内存占用最高的那个
│
├── 2. 是不是segfault
│   dmesg | grep -i "segfault"
│   ls -la /var/lib/systemd/coredump/ 2>/dev/null
│   cat /proc/sys/kernel/core_pattern
│
├── 3. 是不是信号杀死
│   journalctl -u <service> | grep -i "signal\|kill\|term"
│   # SIGTERM(15) = 正常停止 / SIGKILL(9) = 强杀/OOM
│
├── 4. systemd重启限制
│   systemctl show <service> | grep -E "Restart|StartLimit"
│   # 短时间内重启太多次会被systemd禁启动
│
└── 5. 临时处置
    # 内存不够：加swap或限制进程内存
    # 频繁崩溃：检查日志找规律
    # 详见 references/incident-response.md
```

## 症状6：网络不通 / 丢包 / 延迟高

```
ping不通 / 延迟高 / 丢包
│
├── 1. 物理层
│   ip link show                 # 网卡UP/DOWN
│   ethtool eth0                 # Link detected: yes/no
│
├── 2. 数据链路层
│   ip neigh show                # ARP表
│   arping -I eth0 <目标IP>
│   # ARP问题：IP冲突、ARP Flux（双网卡）
│
├── 3. 网络层
│   ping -c10 <目标IP>           # 丢包率
│   ping -c10 -s 1472 -M do <目标IP>  # MTU问题
│   traceroute <目标IP>          # 路径追踪
│   mtr -n -c 30 <目标IP>        # 持续追踪+丢包统计
│   ip route get <目标IP>        # 路由表确认
│   # 云环境注意：安全组可能在网络层静默丢包
│
├── 4. 传输层
│   ss -tnp | grep <目标IP>      # 有没有连接
│   ss -tn state time-wait | wc -l  # TIME_WAIT堆积
│   nstat -az | grep -i retrans    # TCP重传率
│
├── 5. DNS层（最容易被忽略）
│   nslookup <域名>
│   dig <域名> +trace
│   # 症状：IP直连正常但域名访问失败
│
└── 6. tcpdump抓包
    tcpdump -nn -i eth0 host <IP> and port <PORT> -c 50
    # 详见 references/network-layered-troubleshooting.md
```

## 症状7：数据不一致 / 主从延迟

```
从库数据落后 / 复制中断 / 数据不一致
│
├── 1. 复制状态
│   # MySQL主从
│   SHOW SLAVE STATUS\G          # IO线程/SQL线程状态、Seconds_Behind_Master
│   # MySQL MGR
│   SELECT * FROM performance_schema.replication_group_members;
│   SELECT * FROM performance_schema.replication_group_member_stats\G
│   # Redis
│   INFO replication             # master_link_status / master_last_io_seconds_ago
│
├── 2. 复制中断原因
│   # MySQL常见：主键冲突(1062)、记录不存在(1032)、DDL冲突
│   # 解决：跳过错误或重建从库
│   STOP SLAVE; SET GLOBAL SQL_SLAVE_SKIP_COUNTER=1; START SLAVE;
│
├── 3. 延迟原因
│   # 从库性能差、大事务、网络带宽不足、从库有读写压力
│   # 检查从库负载：top/iostat
│   # 检查大事务：SHOW PROCESSLIST
│
└── 4. 数据一致性校验
    # pt-table-checksum / pt-table-sync（MySQL）
    # redis-check（Redis）
```
