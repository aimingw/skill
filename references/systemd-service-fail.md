# systemd 服务启动失败 - 调试方法

systemd 启动服务失败时，错误信息经常被 systemd 吞掉或模糊化。
本文提供从模糊报错到精确定位的完整流程。

## 标准排查流程

```bash
# 第1步：看服务状态（快速概览）
systemctl status <service>
# 关注：
#   Active: active (running) / failed / inactive (dead) / activating (start)
#   Main PID: 1234 (code=exited, status=1/FAILURE) / (code=killed, signal=KILL)
#   Process: 1234 ExecStart=... (code=exited, status=203/EXEC)

# 第2步：看详细日志（最重要）
journalctl -u <service> --since "30 min ago" --no-pager
# 或看最近的50行
journalctl -u <service> -n 50 --no-pager

# 第3步：看服务定义
systemctl cat <service>
# 看 ExecStart / Type / User / Environment / LimitNOFILE 等

# 第4步：直接运行二进制（绕过systemd，看真实报错）
# 这是最关键的一步！systemd会模糊化错误
```

## exit code 解读

| status | 含义 | 常见原因 |
|--------|------|---------|
| 0 | 正常退出 | Type=oneshot 的服务正常完成 |
| 1 | 通用错误 | 配置错误、依赖缺失 |
| 2 | 误用命令行 | 参数不对 |
| 126 | 没有执行权限 | chmod +x |
| 127 | 命令不存在 | 路径写错/没安装 |
| 130 | Ctrl+C中断 | 人工取消 |
| 137 | 被SIGKILL(9) | OOM Killer 或管理员 kill -9 |
| 139 | 段错误 | 程序bug / 依赖库版本不对 |
| 143 | 被SIGTERM(15) | 正常停止 |
| 203 | EXEC格式错误 | 二进制路径不对/shebang错误/架构不匹配 |

## 常见故障模式

### 模式1：Type=notify 启动卡住

**症状：** `systemctl start xxx` 卡住不动，状态一直是 `activating (start)`，但进程其实已经启动了。

**原因：** systemd 设置了 `Type=notify`，它等待服务通过 `sd_notify()` 发送 "READY=1" 消息才认为启动完成。如果服务不支持 sd_notify 或没有正确调用，systemd 就永远等下去。

**典型案例：**

| 服务 | 原因 | 修复 |
|------|------|------|
| Keepalived | 只有有 vrrp_instance 时才发 notify | 补充完整配置 |
| Redis | 需要 `supervised systemd` 配置 | 在 redis.conf 中设置 |
| 自定义服务 | 程序不支持 sd_notify | 改 Type=simple 或 Type=forking |

**调试：**
```bash
# 看进程在不在
ps aux | grep <service>
# 进程在 = Type=notify 问题

# 解决：临时改 Type
systemctl edit <service>
# 在覆盖配置中写：
[Service]
Type=forking
# 或
Type=simple
```

### 模式2：ExecStart 路径不对

**症状：** status=203/EXEC

```bash
# 检查 ExecStart 指定的路径
systemctl cat <service> | grep ExecStart
# 然后确认文件存在
ls -l /usr/sbin/nginx    # 路径要对
file /usr/sbin/nginx     # 架构要对（不能在x86上跑ARM二进制）
```

### 模式3：权限/用户问题

```bash
# 看服务以什么用户运行
systemctl cat <service> | grep -E "User|Group"

# 检查文件权限
ls -l <二进制>
ls -l <配置文件>
ls -ld <数据目录>

# 手动用该用户测试
sudo -u <user> /usr/sbin/nginx -t
```

### 模式4：端口冲突

```bash
# 看谁占了端口
ss -tlnp | grep <PORT>
lsof -i :<PORT>

# 典型场景：
# - Nginx 默认 server 块和新配置冲突（Rocky）
# - 旧进程没退出，新进程绑不上
# - 两个服务配了同一个端口

# 解决
pkill <旧进程>
sleep 1
systemctl start <service>
```

### 模式5：StartLimit 频繁重启被禁

**症状：** 服务反复重启几次后，systemctl start 报 "Start request repeated too quickly"。

```bash
# 查看重启限制
systemctl show <service> | grep -E "StartLimit|Restart"
# StartLimitBurst=5    -> 5次
# StartLimitIntervalSec=10  -> 10秒内

# 解决1：清除失败计数
systemctl reset-failed <service>

# 解决2：临时调大限制
systemctl edit <service>
[Unit]
StartLimitIntervalSec=0    # 不限制
```

### 模式6：配置文件依赖目录不存在

```bash
# 服务日志报：can't open log file /var/log/xxx/
# 但配置文件里写的是这个路径
ls -ld /var/log/xxx/
# 不存在 -> 创建
mkdir -p /var/log/xxx/
chown <user>:<group> /var/log/xxx/
```

## 直接运行二进制的调试技巧

systemd 会模糊化错误信息。最有效的调试方法是绕过 systemd，直接运行：

```bash
# Nginx：前台运行
/usr/sbin/nginx -g "daemon off;"

# MySQL：前台运行
mysqld --user=mysql --console

# Redis：前台运行
redis-server /etc/redis/redis.conf --daemonize no

# Keepalived：不fork前台运行
keepalived -n -l -f /etc/keepalived/keepalived.conf

# 通用：用 strace 看系统调用
strace -f /usr/sbin/<binary> <args> 2>&1 | grep -E "open|connect|bind"
```

这样所有错误信息直接输出到终端，不会被 systemd 过滤。

## 调试 checklist

```bash
# 1. 状态
systemctl status <service>

# 2. 日志
journalctl -u <service> -n 50 --no-pager

# 3. 服务定义
systemctl cat <service>

# 4. 直接运行
/usr/sbin/<binary> <args>

# 5. 权限
ls -l <二进制> <配置> <数据目录>

# 6. 端口
ss -tlnp | grep <PORT>

# 7. 依赖
systemctl list-dependencies <service>

# 8. SELinux
getenforce

# 9. 配置语法
<service> -t  # 或对应的检查命令

# 10. 重置重试
systemctl reset-failed <service>
systemctl start <service>
```
