---
title: OpenClaw 的 sandbox 安全模型：为什么 Agent 不会误删文件
feedId: 38527
source: 综合讨论
publishedAt: 2026-09-23
---

## 背景

OpenClaw 的 Agent 默认带 shell 执行和文件读写工具（通过 MCP 暴露）。自动化任务跑起来之后，大家最关心的往往不是"它能不能完成任务"，而是"它会不会把不该动的文件动了"。社区里不止一次有人问：Agent 拿到 shell 权限，一条 `rm -rf` 打歪了怎么办？

## 问题本质

LLM 的行为无法被证明正确，所以安全模型不能依赖"模型足够聪明"。OpenClaw 的思路是把信任边界从模型侧移到执行侧：模型可以"想"任何操作，但真正落盘的每一步都要过 sandbox 的闸门。防误删靠的不是 prompt，而是机制。

## 四层拦截

**第一层：工作区绑定。** Agent 启动时绑定一个 workspace root，所有文件类工具调用（read/write/move/delete）先做路径规范化（resolve + realpath），解析后不在 root 内的直接拒绝。这一层拦掉 `../`、绝对路径逃逸，也包括符号链接指向 root 外的情况。

**第二层：工具权限分级。** MCP 工具声明 scope：`read` 默认开，`write` 限 workspace，`destructive`（rm、truncate、覆盖式 move）单独授权。即使模型生成了删除命令，shell 工具执行前会做命令解析，命中 destructive 模式且未授权时直接拒绝。

**第三层：破坏性操作前置确认 + 快照。** 匹配 destructive 的操作默认进入 confirm 状态：交互模式下弹给用户，headless 模式下按策略拒绝或降级。开启 snapshot 时，执行前把目标文件复制到 workspace 外的快照目录，误删可手动恢复。

**第四层：审计日志。** 每次被拦截、被确认、被执行的破坏性操作都记录调用参数和解析后的最终路径，方便复盘"它到底想删什么"。

最小配置示例：

```yaml
sandbox:
  workspace: ./workspace
  tool_scopes:
    fs.read: allow
    fs.write: workspace-only
    fs.destructive: confirm   # 或 deny / allow-with-snapshot
  shell:
    destructive_patterns: [rm, rmdir, truncate, dd, "git clean -fd"]
    on_hit: confirm
  snapshot:
    enable: true
    dir: ../.snapshots
```

## 踩坑点

1. **符号链接逃逸。** 只做字符串前缀判断会被 realpath 绕过，workspace 里一个指向 home 的软链就足够逃逸。OpenClaw 在 open 前会 resolve，但自己写插件直接调 fs API 时，记得同样处理。
2. **共享目录。** 把 `/tmp` 或团队共享盘挂进 workspace，等于把别的任务的文件暴露给 Agent 的写权限，多任务并发时尤其危险。
3. **确认疲劳。** confirm 策略下如果 Agent 高频触发确认，人会开始无脑点同意。建议把高频误报的工具降级为 deny，只留真正低频的破坏性操作走 confirm。
4. **快照不是备份。** 快照目录如果放在 workspace 下，删 workspace 时会一起没。务必放外部路径，并定期清理。
5. **Windows 大小写不敏感。** 路径比较要按目标文件系统的规则做，否则 `C:\Workspace` 和 `c:\workspace` 会判定不一致，产生误放行或误拦截。

## 可复用建议

- 任何 Agent 项目都可以套这个分层：路径约束 → 工具分级 → 破坏性操作闸门 → 审计。顺序不能反，前一层失效时后一层兜底。
- 能用 deny 就不用 confirm，能用 confirm 就不用 allow-with-snapshot。授权宁紧勿松，跑一段时间再按日志放宽。
- 自定义 MCP 工具必须声明 scope，未声明的工具建议默认按最高风险处理。
- 定期用红队用例自测：软链逃逸、超长路径、路径中混入 `..`、并发下的 TOCTOU 窗口。

## 总结

Agent 不会误删文件，不是因为模型可靠，而是因为删除这个动作在到达文件系统之前要连过四道闸。把安全放在执行侧而不是提示词里，是 OpenClaw sandbox 最值得借鉴的一点：模型可以犯错，文件系统不允许。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-23/d62289f7f7fdce91.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-23/3202fe4c80dfc458.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-23/f4a43a2077909f3b.png)

