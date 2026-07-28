# Ops Troubleshooting Skill

Linux 运维排障统一入口 skill，适用于 Hermes Agent/OpenClaw。

## 包含内容

- 5步排障方法论（现象收集 -> 影响评估 -> 分层排查 -> 根因定位 -> 修复验证记录）
- 7棵排障决策树（服务起不来/端口不通/响应慢/磁盘满/进程崩溃/网络不通/数据不一致）
- 排障工具速查表（CPU/内存/IO/网络/进程/日志/系统限制）
- 8个生产高频事故模式库
- 事故响应流程 + 复盘模板

## 文件结构

```
SKILL.md                     主文件（方法论 + 决策树索引 + 速查表 + pitfalls + checklist）
references/
├── decision-trees.md            7棵排障决策树
├── tool-reference.md            排障工具速查
├── accident-patterns.md         高频事故模式 + 快速匹配表
├── selinux-silent-block.md      SELinux静默拦截诊断
├── port-not-listening.md        端口不监听排查流程
├── systemd-service-fail.md      systemd服务启动失败调试
├── network-layered-troubleshooting.md  OSI分层网络排障
├── performance-analysis.md      CPU/内存/IO性能分析
├── disk-resource-exhaustion.md  磁盘/inode/fd/conntrack耗尽
└── incident-response.md          事故响应流程 + 复盘模板
```

## 版本

v0.1 - 初始版本
