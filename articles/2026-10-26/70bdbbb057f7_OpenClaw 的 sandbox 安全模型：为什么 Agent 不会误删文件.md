---
title: OpenClaw 的 sandbox 安全模型：为什么 Agent 不会误删文件
feedId: 42637
source: 综合讨论
publishedAt: 2026-10-26
---

## 背景

让 Agent 拿到 shell 和文件读写权限之后，大家最担心的从来不是它写不出代码，而是某一步"想岔了"：把 `rm` 的目标拼错、把"清理缓存"理解成清空整个目录、或者对值为空的路径变量执行递归删除。OpenClaw 的 sandbox 就是为这类场景设计的——**默认模型一定会犯错，然后用工程手段把错误的代价限制住**。

## 问题：误删是怎么发生的

复盘社区里的几次"惊魂"，路径基本一致：

1. 模型输出的路径来自上下文拼接，遇到符号链接或相对路径时，和真实磁盘位置不一致；
2. 删除命令往往裹在一段正常任务里，人工审阅时容易被长输出淹没；
3. 执行即时生效，没有中间态，错了就是错了。

所以防误删不是"在提示词里多写一句小心点"，而是要在执行链路上设卡。

## OpenClaw 的四层防线

**第一层：工作区监狱（path jail）。** 所有文件类工具调用先做路径规范化（resolve 符号链接、折叠 `..`），再检查是否落在 workspace root 内。越界直接拒绝，工具本体根本不会执行。

**第二层：工具级能力声明。** 每个 Agent profile 和每个 MCP server 都要声明可用能力，比如 `fs:read` / `fs:write` / `fs:delete` / `net:*`，没声明的调用在调度层就被拦下。给一个只做代码审查的 Agent 配 `fs:delete`，本身就是配置错误。

**第三层：破坏性操作识别 + 确认门。** OpenClaw 对删除、覆盖、批量重命名做模式识别，命中后默认进入 dry-run：先产出"将被影响的路径列表 + diff"，等人确认。确认门可以按 profile 关闭，但默认开启。

**第四层：快照与回滚。** 执行破坏性操作前，对目标子树做轻量快照（hardlink 方式，不复制内容），失败可一键回滚。它不阻止错误，但把错误变成可撤销的。

配置大致是：

```yaml
sandbox:
  root: ./workspace
  allow_symlink_escape: false
  capabilities: [fs:read, fs:write]
  destructive:
    confirm: true
    snapshot: true
```

## 踩坑点

- **符号链接逃逸**：workspace 里如果有人手动 `ln -s /` 过，路径规范化后会指向监狱外。保持 `allow_symlink_escape: false`，并把 workspace 当生成物目录，别手工塞软链。
- **模型改写绝对路径**：某些任务里模型会"好心"把相对路径补全成绝对路径，表现为频繁触发越界拒绝。正确解法是在系统提示里约定"所有路径相对 workspace root"，而不是放开监狱。
- **图省事关掉确认门**：批量任务跑久了嫌确认烦，一关，前三层就只剩事后回滚兜底。建议按 profile 区分：只读探索类可自动，写操作类永远留门。
- **并行任务共享 workspace**：两个 Agent 同时做清理，快照和确认会互相踩。给每个任务独立子目录，把授权粒度做到最小。

## 可复用建议

- 把 sandbox 当默认态：新 Agent 一律从最小能力起步，报权限错误再按需扩，而不是反过来。
- 破坏性操作的确认输出要"人能看懂"：路径列表 + diff，远比一段原始日志有用。
- 快照成本很低、恢复价值极高，没有理由关。
- 定期用一条故意写错的删除指令做演练，验证三层拦截真的在工作，而不是只在文档里工作。

## 总结

OpenClaw 防误删靠的不是让模型更聪明，而是默认它不聪明：路径进监狱、能力要声明、删除先预览、出错可回滚。四层防线单独看都不复杂，叠起来之后，"Agent 误删文件"就从一场灾难降级为一次可撤销的操作。这也是自动化任务敢跑长、敢无人值守的真正前提——安全模型决定的不是 Agent 能做什么，而是你敢让它做什么。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-26/e8b4faac45028a5a.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-26/a1a1217083a9c5c3.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-26/e14d542abdb38ee6.png)

