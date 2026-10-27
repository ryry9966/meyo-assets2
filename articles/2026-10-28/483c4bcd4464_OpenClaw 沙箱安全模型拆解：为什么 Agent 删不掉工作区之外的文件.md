---
title: OpenClaw 沙箱安全模型拆解：为什么 Agent 删不掉工作区之外的文件
feedId: 42790
source: 综合讨论
publishedAt: 2026-10-28
---

## 背景

给 Agent 接上 shell 工具之后，大家真正担心的从来不是“它能不能干活”，而是“它会不会手滑”。LLM 的输出本质上不可靠：路径拼错、把相对路径当绝对路径、清理缓存时递归范围判断失误，这些都不是小概率事件。OpenClaw 对此的思路很明确：**不指望模型自觉，把安全约束做在运行环境层**。

## 问题

说一个我们内部做过的演练。让 Agent 清理构建缓存，它生成了 `rm -rf $BUILD_DIR/*`，而变量恰好为空，命令退化成了 `rm -rf /*`。当时 Agent 跑在沙箱里，删掉的是容器内的文件，演练以“预期内失败”收场。但如果这条命令是直接在宿主机 exec 出去的，结果就完全不同了。事故模型很简单：**只要存在一条路径能让模型输出直达宿主文件系统，删除就是时间问题。**

## OpenClaw 的分层做法

1. **工作区边界**：文件类工具（read/write/edit）默认被限制在 workspace 目录内，越界路径在工具层直接拒绝，根本不会进入模型的上下文反复试探。
2. **Docker 沙箱**：exec 类命令跑在独立容器里，sandbox mode 控制哪些 Agent 进沙箱（all / non-main / off）。容器只挂载 workspace，能只读就只读。
3. **exec approvals**：命令先过审批闸门。`git status`、`ls`、`pytest` 这类低风险命令进 allowlist 自动放行；`rm`、`dd`、`mkfs`、`find -delete` 进 denylist 或强制人工确认。
4. **tool policy**：按 Agent 和渠道裁剪工具面，夜间无人值守的会话干脆不给 exec。
5. **提示词只是软约束**：system prompt 里写“不要删文件”属于行为引导，不是安全边界。真正的强制力在前三层。

配置示意（键名以你当前版本文档为准）：

```yaml
agents:
  defaults:
    sandbox:
      mode: non-main
tools:
  exec:
    approvals:
      default: ask
      allow: ["git *", "ls *", "pytest *"]
      deny: ["rm -rf /*", "mkfs*", "dd *"]
```

## 踩坑点

- **图省事把整个 `$HOME` 挂进沙箱**：边界等于没做，容器只是心理安慰。
- **挂载卷默认 rw**：只读数据一律 `ro`，写权限按需单独开。
- **allowlist 过宽**：`rm *`、`bash -c *` 等于把闸门焊死在开位。另外注意 git hook、npm script 这类“被放行的命令会间接拉起任意命令”的执行链。
- **容器内外路径不一致**：相对路径的解析点不同，出问题后审计日志对不上号，排查很痛苦。
- **配置漂移**：以为非 main 会话默认进沙箱，实际 mode 是 off；升级覆盖配置后务必复查。

## 可复用建议

- 默认策略用“**默认 ask + 白名单放行**”，而不是“默认允许 + 黑名单拦截”，黑名单永远追不完。
- 最小挂载、只读优先，写权限是例外而非常态。
- 每次改完配置，跑一次**注定失败的删除演练**（比如让 Agent 删一个标记目录），亲眼确认边界生效。
- 无人值守场景优先收窄工具面，多几次人工确认不丢人。

## 总结

“Agent 不误删文件”不是模型品德问题，而是工程问题：把不可信的模型输出放进可信的执行环境，用工作区边界、容器隔离、命令审批三层硬约束兜底，提示词只做补充。OpenClaw 的好处是这些层全部声明式配置，调整成本是分钟级的。建议每个部署过 Agent 的人都做一次删除演练——你验证过的边界才叫边界，配置文件里写了的不算。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-28/d7cdc49cc6976580.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-28/bdb0d4c0908ccc53.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-28/331a03a11f08db17.png)

