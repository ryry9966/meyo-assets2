---
title: OpenClaw 的 sandbox 安全模型：为什么 Agent 不会误删文件
feedId: 42796
source: 综合讨论
publishedAt: 2026-10-28
---

# 背景

让 Agent 直接操作文件系统，最担心的从来不是它不会干活，而是它“太会干活”——一条想当然的 `rm -rf`、一次路径理解偏差，就可能把项目目录清空。OpenClaw 的基本立场是：不指望模型自律，而是把容错做进执行层。这篇拆一下 OpenClaw sandbox 的分层安全模型，以及为什么在默认配置下，Agent 删不掉工作区之外的文件。

# 问题

实际使用中，“误删”通常来自三类情况：

1. **模型幻觉**：把相对路径当成绝对路径，或把 `/tmp/xxx` 记成 `/xxx`。
2. **提示注入**：网页、日志、第三方文档里夹带的指令，诱导 Agent 执行“清理”类命令。
3. **命令链逃逸**：规则拦了 `rm -rf`，Agent 换成 `find -delete`、`xargs rm`，或塞进 `bash -c` 里绕过去。

三者的共同点：靠 prompt 写“请小心操作”没用。错误发生在生成层，防线必须落在执行层。

# 做法：四层防线

OpenClaw 的 sandbox 不是单点开关，而是分层叠加：

1. **视野隔离（workspace root）**：文件工具默认把 Agent 的可见范围限制在 workspace 内，路径解析先做 canonicalize；工作区内的 symlink 指向外部时直接拒绝，防住“在目录里放个软链接通向 /etc”这类逃逸。
2. **能力分级（tool permission)**：读、写、执行是三种独立权限。默认只开放 workspace 内读写；shell 工具默认关闭，开启后走独立的执行策略。
3. **命令策略（command policy）**：对 shell 命令做 argv 语义解析而非正则匹配，能识别 `find -delete`、重定向覆盖等变体；命中不可逆操作模式直接拦截，而不是提示模型“别这么干”。
4. **快照与审批（snapshot & approval）**：不可逆写操作（覆盖、删除、批量移动）默认走审批门，或执行前自动打快照可回滚；所有工具调用进审计日志，出事后能完整回放。

配置形态大致如下（字段名以当前版本文档为准）：

```yaml
sandbox:
  root: ./workspace
  allowSymlinkEscape: false
  permissions:
    fs: read-write
    shell: ask        # 每次执行需人工确认
  policy:
    denyIrreversible: true
    snapshotBeforeWrite: true
```

# 踩坑点

- **shell 权限开成 always-allow**：文件工具的所有防护对裸 shell 无效，等于只锁了正门。shell 必须走命令策略。
- **容器里用 root 跑 Agent**：workspace 隔离挡得住路径，挡不住 root 权限下的其他破坏面。固定用非 root 用户，系统路径只读挂载。
- **Deny 正则写得太具体**：拦得住 `rm -rf`，拦不住 `rm -r -f`。优先用语义解析的 policy，别自己手写黑名单。
- **审批疲劳**：图省事把 `ask` 全改成 `allow`，审批门形同虚设。正确做法是把高频安全操作加白名单，而不是全局放行。
- **把快照当兜底就关隔离**：回滚只能救数据，救不了越权读走的内容。

# 可复用建议

- 默认最小权限，按需放行，放行动作要留痕。
- 安全策略放执行层；prompt 里的“注意安全”只当提示，不当防线。
- 上线前跑一段红队脚本自测：删工作区外文件、建外指软链、命令链绕过，预期全部被拦截或要求审批，任何一条漏网都要回到策略层修。
- 审计日志落盘并定期抽查，别等出事才翻。

# 总结

Agent 不会误删文件，不是因为模型足够聪明，而是因为系统不允许它犯这种错。OpenClaw sandbox 的思路可以压缩成四句话：视野收窄、能力分级、不可逆设卡、留快照和审计。模型会犯错是常态，工程的价值在于让错误的后果可承载、可回放、可恢复。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-28/2b50a07cfd88af0b.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-28/78aa43225a067ce7.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-28/fc6ab2508e8a2d8f.png)

