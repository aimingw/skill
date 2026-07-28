# SELinux 静默拦截 - 完整诊断与修复

SELinux 是 Rocky/RHEL/CentOS 系统上最容易导致"诡异故障"的原因。
它最坑的地方不是拦截本身，而是**拦截了但不留日志**。

## 核心特征：静默拦截（dontaudit）

SELinux 有两类规则：
- **allow/deny**: 正常允许/拒绝，拒绝会写 audit 日志
- **dontaudit**: 静默丢弃，**不写任何日志**

dontaudit 的设计初衷是"减少日志噪音"——对已知的无害拒绝不记录。
但副作用是：当这个拒绝恰好是故障原因时，你**完全看不到任何线索**。

## 三种典型拦截模式

### 模式1：端口绑定型（服务启动了但端口不监听）

**症状：**
- `systemctl start xxx` 返回成功
- 进程在运行（`ps aux` 能看到）
- 但 `ss -tlnp` 看不到端口
- 日志里没有任何报错
- `dmesg | grep selinux` / `ausearch -m avc` 都没有记录

**典型案例：MySQL MGR 的 33061 端口**

```
# MGR 配置正确，START GROUP_REPLICATION 执行了
# 但 ss -tlnp | grep 33061 没输出
# 对端 telnet 也不通
# 日志干净得像什么都没发生过
```

**诊断流程：**

```bash
# 1. 确认端口确实没监听
ss -tlnp | grep 33061
# 空输出

# 2. 本机自连测试
timeout 3 bash -c 'echo > /dev/tcp/127.0.0.1/33061 && echo OK' 2>&1
# Connection refused

# 3. 检查 SELinux
getenforce
# Enforcing  <- 嫌疑很大

# 4. 关闭 SELinux 验证
setenforce 0
systemctl restart <service>
ss -tlnp | grep 33061
# 端口出现了 -> 100% 是 SELinux
```

**修复（永久，不关闭SELinux）：**

```bash
# 方法1：放行自定义端口（推荐）
semanage port -a -t mysqld_port_t -p tcp 33061
# 如果 semanage 未安装：
dnf install -y policycoreutils-python-utils   # Rocky/RHEL 9
apt install -y policycoreutils                  # Ubuntu（如果装了SELinux）

# 方法2：临时关闭SELinux（仅学习/测试环境）
setenforce 0
sed -i 's/^SELINUX=enforcing/SELINUX=disabled/' /etc/selinux/config

# 方法3：生成并加载自定义策略模块
ausearch -m avc -ts recent -c mysqld 2>/dev/null | audit2allow -M mymysql
semodule -i mymysql.pp
# 注意：如果 dontaudit 规则在生效，ausearch 可能没有输出
# 需要先临时关闭 dontaudit：
# semodule -DB    # 关闭所有 dontaudit
# 然后复现问题，再 ausearch
# 事后恢复：semodule -bD
```

### 模式2：网络连接型（服务能连本地但连不上远程）

**典型案例：Nginx 反向代理 502**

```
# Nginx 配置了 proxy_pass 到后端
# 直接 curl 后端IP:端口 能通
# 但通过 Nginx 访问返回 502
# Nginx error.log: connect() to IP:port failed (13: Permission denied)
```

**诊断：**

```bash
# 1. 确认后端直连正常
curl -I http://<后端IP>:<PORT>
# 200 OK -> 后端没问题

# 2. 看 Nginx 错误日志
tail /var/log/nginx/error.log
# connect() failed (13: Permission denied)
# 13 = EACCES = 权限被拒 -> SELinux 拦截

# 3. 检查 SELinux 布尔值
getsebool httpd_can_network_connect
# httpd_can_network_connect -> off  <- 就是这个

# 4. 临时验证
setenforce 0
curl -I http://localhost
# 200 OK -> 确认是 SELinux
```

**修复：**

```bash
# 放行 Nginx/Apache 发起网络连接
setsebool -P httpd_can_network_connect on
# -P = 持久化，重启后依然生效

# 其他相关布尔值（按需）
getsebool -a | grep httpd
# httpd_can_network_connect_db -> off  (连数据库)
# httpd_can_network_memcache -> off    (连memcache)
# httpd_enable_homedirs -> off         (访问用户家目录)
```

**跨发行版注意：**
| 系统 | SELinux | 此问题是否存在 |
|------|---------|---------------|
| Rocky/RHEL/CentOS 9 | 默认 Enforcing | 是，默认 off |
| Ubuntu/Debian | 默认不装 SELinux | 不存在 |
| OpenEuler | 可能有 SELinux | 取决于配置 |

### 模式3：文件访问型（能读不能写/不能执行）

**典型案例：服务无法写入数据目录**

```
# 服务以 mysql 用户运行
# /data/mysql/ 目录权限是 mysql:mysql 755
# 但 mysqld 就是写不进去
# 日志报：Permission denied
```

**诊断：**

```bash
# 1. 普通权限检查
ls -ld /data/mysql/
# drwxr-xr-x mysql mysql -> 普通权限没问题

# 2. SELinux 上下文检查
ls -Zd /data/mysql/
# unconfined_u:object_r:default_t:s0
# default_t = SELinux 不认识这个路径，不允许服务访问

# 3. 对比标准路径的上下文
ls -Zd /var/lib/mysql/
# system_u:object_r:mysqld_db_t:s0
# 这才是正确的上下文
```

**修复：**

```bash
# 方法1：修改安全上下文（推荐）
semanage fcontext -a -t mysqld_db_t "/data/mysql(/.*)?"
restorecon -Rv /data/mysql/

# 方法2：临时验证
setenforce 0
systemctl restart mysqld
# 能写了 -> 确认是 SELinux
setenforce 1

# 方法3：通用但不安全
chcon -Rt default_t /data/mysql/    # 不推荐，重启后可能丢失
```

## 速查：SELinux 排障命令清单

```bash
# 状态
getenforce                              # Enforcing / Permissive / Disabled
sestatus                                # 详细状态 + 策略版本

# 布尔值（功能开关）
getsebool -a                            # 所有布尔值
getsebool httpd_can_network_connect     # 查单个
setsebool -P httpd_can_network_connect on  # 永久开启

# 端口标签
semanage port -l                        # 所有端口标签
semanage port -l | grep mysqld          # MySQL相关端口
semanage port -a -t mysqld_port_t -p tcp 33061  # 添加端口

# 文件上下文
ls -Z /path/to/file                     # 查看文件上下文
semanage fcontext -l                    # 所有上下文规则
semanage fcontext -a -t mysqld_db_t "/data/mysql(/.*)?"  # 添加规则
restorecon -Rv /data/mysql/             # 应用上下文

# 日志排障
ausearch -m avc -ts recent              # 最近的 AVC 拒绝
ausearch -m avc -c mysqld              # 某进程的 AVC
sealert -a /var/log/audit/audit.log    # 详细分析（需 setroubleshoot）
journalctl -t setroubleshoot           # 人类可读的 SELinux 告警

# dontaudit 处理
semodule -DB                            # 临时关闭所有 dontaudit（排障用）
# 复现问题后：
ausearch -m avc -ts recent              # 现在能看到被静默的拒绝了
semodule -bD                           # 恢复 dontaudit

# 策略模块
audit2allow -a -M mypolicy             # 从 AVC 日志生成策略
semodule -i mypolicy.pp                 # 加载策略
```

## 判断口诀

```
服务能启动、端口不监听、日志全干净、检查SELinux。
```

```
Permission denied + 普通权限没问题 = 查SELinux上下文。
```

```
Nginx 502 + 后端直连正常 = 查 httpd_can_network_connect。
```
