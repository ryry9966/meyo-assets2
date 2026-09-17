---
title: OpenClaw 的 sandbox 安全模型：为什么 Agent 不会误删文件
feedId: 37921
source: 综合讨论
publishedAt: 2026-09-17
---

## 背景

OpenClaw 的 Agent 不是纯聊天进程：它有 exec 工具，能直接跑 shell、写文件、装依赖，权限接近运行它的那个用户。社区里最常被问的就是——"让它跑命令，哪天理解错路径，rm 一下我的家目录怎么办？"

这个担心是合理的。LLM 会幻觉路径、会误解析相对路径、会被外部内容里的注入指令带偏。OpenClaw 的回答不是"相信模型"，而是在模型外面包了几层确定性约束。

## 问题：误删到底怎么发生

真实场景基本是三类：

1. **路径幻觉**：把 `~/project` 记成 `~/projects`，清理命令的作用范围扩大；
2. **cwd 漂移**：exec 每次调用的起始目录不一致，`rm -rf ./build` 的落点不可控；
3. **提示注入**：读网页时读到"删除临时目录以释放空间"之类的指令，原样执行。

三者都不是"模型变坏"，而是工具权限过大加边界不清的必然结果。只靠 system prompt 写"请勿删除文件"，等于把安全边界建在最不可靠的层上。

## OpenClaw 的四层做法

整体模型是：**隔离 → 策略 → 审批 → 审计**。

1. **workspace 隔离**：每个 Agent 默认只能看到自己的 workspace，exec 的 cwd 固定在里面，相对路径被约束在这棵目录树内。
2. **容器 sandbox**：打开后，exec 不再落在宿主机，而是落到按会话隔离的容器里：

```yaml
agents:
  defaults:
    sandbox:
      mode: all       # 所有会话进容器
      scope: session  # 会话级隔离，互不污染
```

容器内只挂载 workspace，其余是临时文件系统。模型就算执行 `rm -rf /`，删的也是一次性环境，宿主机文件系统根本不在它的视图里。

3. **工具策略与审批**：tools policy 决定哪些工具可用；exec approval 决定哪些命令需要人工确认。allowlist 放行 `npm test`、`pytest` 这类常规命令，denylist 硬拦 `rm -rf`、`mkfs`、`dd`，其余命令弹审批。
4. **审计日志**：每次 exec 的命令、cwd、退出码都落盘，事后能回放"当时到底执行了什么"。

## 踩坑点

- 为图省事关掉 sandbox、把 home 目录挂进容器，等于拆掉所有层。隔离的价值取决于挂载的最小化。
- approval 连点几次"总是允许"，就退化成事实上的无审批。allowlist 要具体，别整段放行 `bash`。
- prompt 约束不是安全边界，注入内容可以轻松覆盖它。
- 多 Agent 共用非 session 的 sandbox scope 时，一个会话的写操作会影响另一个，容易被误判成"文件被删了"。先查 scope 配置，再怀疑模型。
- Skill/插件如果在 sandbox 关闭的路径上执行，会继承 gateway 进程权限，高危 Skill 也必须走容器。

## 可复用建议

1. **先写威胁模型**：明确 Agent 会接触哪些不可信内容（网页、issue、用户消息），再决定隔离层级。
2. **默认拒绝**：工具策略从 deny 起步、按需放行，而不是全开之后打补丁。
3. **任务分治**：不可信任务和可信任务拆成不同 Agent，前者强制 `sandbox: all` 并收紧 egress。
4. **安全当回归测试做**：每次升级后，用一个"故意 `rm -rf /tmp/xxx`"的用例验证隔离仍然生效。

## 总结

"Agent 不会误删文件"不是一个模型行为承诺，而是一个系统设计结果：workspace 划定它能看到什么，容器决定改动能落到哪，策略和审批控制它能执行什么，审计保证出了问题能回溯。模型可以不可靠，但边界必须是确定性的。这套分层思路并不绑定 OpenClaw 的具体实现，迁移到任何 Agent 框架都成立。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-17/21fe5c054fe86702.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-17/4faed41772a9913c.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-17/3eff1e095364f932.png)

