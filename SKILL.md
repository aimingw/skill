---
name: ops-troubleshooting
description: "Use when 生产服务故障、端口不通、性能瓶颈、磁盘满、进程崩溃、网络异常等运维排障场景。提供5步排障方法论、按症状的决策树、工具速查和高频事故模式库。Don't use for 纯代码级bug调试（用systematic-debugging）。"
version: 1.0.0
author: Hermes Agent
license: MIT
category: devops
metadata:
  hermes:
    tags: [troubleshooting, ops, sre, debugging, root-cause, incident-response, performance]
    related_skills: [systematic-debugging, nginx-reverse-proxy, mysql-mgr, dns-master-slave, keepalived-lab, rhel-source-build, linux-ops-teach]
---

# Ops Troubleshooting - 运维排障统一入口

## Overview

运维排障的"总入口"。遇到线上故障时先来这里找方向，不替代各技术栈的专项排障（Nginx/MySQL/DNS等有各自的skill）。

核心价值三件事：
1. 给一个排障方法论框架（5步法），避免凭直觉乱猜
2. 按症状快速定位排查方向（决策树），省去"不知道先查什么"的时间
3. 汇总生产高频事故模式，遇到时直接套经验

## When to Use

Use this skill when:
- 服务起不来 / systemctl start 失败或卡住
- 端口不通 / 连接拒绝 / 连接超时
- 响应慢 / 负载高 / CPU飙高 / 内存耗尽
- 磁盘满 / inode耗尽 / fd耗尽 / No space left
- 进程挂了 / OOM / segfault / zombie
- 网络不通 / 丢包 / 延迟高 / DNS解析失败
- 主从不同步 / 复制延迟 / 数据不一致
- 生产事故 / 应急响应 / 故障复盘

Don't use for:
- 纯代码级bug调试（用 `systematic-debugging`，它有源码阅读和回归测试流程）
- 单一技术的配置问题（用对应专项skill：`nginx-reverse-proxy`、`mysql-mgr`、`dns-master-slave`）
- 源码编译问题（用 `rhel-source-build`）

**协作原则：本skill负责"找方向"，专项skill负责"挖细节"。**

## The 5-Step Troubleshooting Method

软件调试可以慢慢复现、读源码、写测试。运维排障不行——生产挂了你只有几分钟。
所以核心区别是：**先止血再查因，先看面再看点。**

### Step 1: 现象收集（What happened）

30秒内搞清楚"什么挂了、影响了谁"。

| 问题 | 查什么 | 命令 |
|------|--------|------|
| 什么服务？什么症状？ | 服务状态、端口、进程 | `systemctl status xxx` / `ss -tlnp` / `ps aux` |
| 从什么时候开始的？ | 系统日志时间线 | `journalctl --since "30 min ago" --no-pager` |
| 影响了谁？ | 监控告警、用户反馈 | 看Grafana/Zabbix大盘、问业务方 |

**关键原则：不要只看一个维度。** 服务挂了可能是磁盘满了，磁盘满了可能是日志没轮转，日志没轮转可能是logrotate配置错了。沿着链条往上追。

**完成标准：** 能用一句话描述"什么服务在什么时间出现了什么症状，影响了谁"。

### Step 2: 影响评估（How bad is it）

决定响应级别，该不该立刻止血。

| 级别 | 标准 | 响应动作 |
|------|------|----------|
| P0 | 核心业务全挂/数据丢失 | 立刻止血：回滚/重启/切备/降级 |
| P1 | 部分功能不可用 | 5分钟内到现场，先止血再查因 |
| P2 | 性能劣化 | 评估趋势，决定是否紧急处理 |
| P3 | 告警但未影响用户 | 记录工单，计划窗口处理 |

**止血优先原则：** P0/P1先做能让业务恢复的操作，不要在事故中搞根因分析。根因留到事后复盘。

**例外：** 怀疑被入侵或数据被删，不要重启服务（可能破坏证据）。先隔离机器，保留现场，进入安全应急流程（见 `references/incident-response.md`）。

**完成标准：** 确定了事故级别Pn，决定了是先止血还是先查因。

### Step 3: 分层排查（Where is it broken）

用OSI模型从下往上定位故障层。**排查顺序口诀：先通后服务，先下后上。**

```
第7层 应用层    服务配置对不对？代码有没有bug？       -> cat配置 / tail日志 / curl测试
第4层 传输层    端口在不在？防火墙挡没挡？           -> ss -tlnp / iptables / firewalld
第3层 网络层    IP通不通？路由对不对？               -> ping / traceroute / ip route
第2层 数据链路层 MAC对不对？ARP表正不正？           -> arp -a / ip neigh / ethtool
第1层 物理层    网线插没插？网卡down没down？         -> ip link / ethtool
```

具体操作命令和跨机排查模板见 `references/network-layered-troubleshooting.md`。

遇到具体症状时，按决策树逐项检查，详见 `references/decision-trees.md`。

**完成标准：** 故障定位到某一层（如"网络层路由问题"或"应用层配置错误"），不是停留在"不知道哪的问题"。

### Step 4: 根因定位（Why did it break）

用假设-验证法找到真正的根因，不是表面症状。

```
观察现象 -> 形成假设 -> 设计验证 -> 执行验证 -> 结论
   ^                                              |
   |_______ 如果验证不通过，换假设，回到观察 ________|
```

**三个铁律：**

1. **一次只改一个变量。** 不要同时改配置+重启+清缓存，出了问题你不知道哪个起作用了。

2. **不要停在症状层。**
   - "磁盘满了"不是根因
   - "日志没轮转"也不是根因
   - "logrotate配置里rotate写成了rotete"才是根因
   - 追到能回答"为什么会在这一刻发生"才算到根因

3. **警惕多因一果。** 修了一个症状还在 -> 不要"再试试别的"，回到Step 3重新排查。

常见根因分类和验证方法见 `references/accident-patterns.md`。

**完成标准：** 能说清"因为X导致了Y，验证方法是Z，修复X后Y消失"。

### Step 5: 修复 + 验证 + 记录（Fix, Verify, Document）

**修复原则：**
1. 先备份再改：`cp config config.bak.$(date +%Y%m%d%H%M)`
2. 最小变更：只改根因涉及的部分，不要"顺手优化"
3. 回滚预案：改之前想好怎么回滚

**验证清单：**
```bash
systemctl is-active <service>          # active (running)
ss -tlnp | grep <PORT>                # 端口在听
curl -I http://localhost:<PORT>       # HTTP 200
journalctl -u <service> --since "5 min ago" --no-pager  # 无新error
# 观察至少5分钟，确认没有复发
```

**记录：** 事故恢复后48小时内完成复盘，模板见 `references/incident-response.md`。

**完成标准：** 服务恢复active、功能验证通过、监控指标正常、观察5分钟无复发、复盘文档已写。

## Quick Match Table

看到关键词快速定位。完整版见 `references/accident-patterns.md`。

| 关键词 | 第一步检查 |
|--------|-----------|
| Connection refused | `ss -tlnp \| grep <PORT>` |
| Connection timed out | `ping` + `traceroute` |
| Permission denied | `ls -l` + `getenforce` |
| No space left | `df -h` + `df -i` |
| Too many open files | `ls /proc/<PID>/fd \| wc -l` |
| Address already in use | `ss -tlnp \| grep <PORT>` |
| killed process (dmesg) | `dmesg \| grep killed` |
| 502 Bad Gateway | `curl backend:port` + `getsebool httpd_can_network_connect` |
| 服务能启动、端口不监听、日志全干净 | `getenforce` -> `setenforce 0` 验证 |

## References

| 文件 | 内容 |
|------|------|
| `references/decision-trees.md` | 7棵排障决策树（服务起不来/端口不通/响应慢/磁盘满/进程崩溃/网络不通/数据不一致） |
| `references/tool-reference.md` | 排障工具速查（CPU/内存/IO/网络/进程/日志/系统限制） |
| `references/accident-patterns.md` | 8个高频事故模式 + 快速匹配表 |
| `references/selinux-silent-block.md` | SELinux静默拦截诊断和修复 |
| `references/port-not-listening.md` | 端口不监听完整排查流程 |
| `references/systemd-service-fail.md` | systemd服务启动失败调试 |
| `references/network-layered-troubleshooting.md` | OSI分层网络排障 + tcpdump实战 |
| `references/performance-analysis.md` | CPU/内存/IO性能分析深度指南 |
| `references/disk-resource-exhaustion.md` | 磁盘满/inode/fd/conntrack耗尽 |
| `references/incident-response.md` | 事故响应流程 + 复盘模板 |

## Common Pitfalls

1. **跳层排查。** 不查网络直接猜应用问题。必须从物理层往上逐层排查，不要跳层。

2. **同时改多个东西。** 同时改配置+重启+清缓存，出了问题不知道哪个起作用了。一次只改一个变量。

3. **停在症状层。** "磁盘满了"就清理完走人，不追根因（logrotate配置拼写错误）。同样的问题下周还会发生。

4. **在生产环境做实验性排查。** 需要复现就在测试环境复现。生产环境只做只读检查（cat/grep/ss/journalctl），不做任何修改。

5. **事故中搞根因分析。** P0/P1应该先止血（回滚/重启/切流量），不要在业务不可用时慢慢查根因。根因留到事后复盘。

6. **不看 dmesg。** OOM Killer杀进程、segfault、网卡链路down只在dmesg/kernel日志里有记录，应用日志看不到。

7. **忽略时钟同步。** SSL/Kerberos/JWT认证间歇失败，查了半天密码证书都没问题，其实是时钟漂移。`date` + `chronyc tracking` 应该是常规检查项。

8. **在 Rocky/RHEL 上忘记 SELinux。** 服务能启动、端口不监听、日志全干净 -> 先 `getenforce` 再 `setenforce 0` 验证。口诀：服务能启动、端口不监听、日志全干净、检查SELinux。

9. **用 strace -p 追踪高QPS进程。** strace 会让目标进程性能下降数倍甚至数十倍。生产环境只在低QPS或短暂使用，或用 `-c` 只做统计不逐条追踪。

10. **不写复盘文档。** 修完就走人，同样的故障下次还会发生。48小时内必须完成复盘，沉淀经验。

## Verification Checklist

排障完成后的验证清单：

- [ ] 服务状态恢复：`systemctl is-active <service>` = active (running)
- [ ] 端口在监听：`ss -tlnp | grep <PORT>` 有输出
- [ ] 功能验证通过：`curl` / 对应功能测试命令返回正常
- [ ] 日志无新错误：`journalctl -u <service> --since "5 min ago"` 无error/warning
- [ ] 监控指标恢复正常（Grafana/Zabbix 确认）
- [ ] 观察至少5分钟，确认没有复发
- [ ] 根因已定位（不是停在症状层）
- [ ] 修复措施已实施（根因修复，不只是止血）
- [ ] 配置文件已备份（.bak 文件存在）
- [ ] 如有回滚操作，确认回滚完成且业务正常
- [ ] 复盘文档已写（48小时内）
- [ ] 预防措施已列入待办（监控告警/流程改进）
