---
title: OpenClaw 会话隔离实战：让子 Agent 干脏活，主会话保持干净
feedId: 42166
source: 综合讨论
publishedAt: 2026-10-21
---

## 背景

OpenClaw 里每个 agent 的会话是独立的 JSONL 文件（`~/.openclaw/agents/<agentId>/sessions/` 下），主会话保存着你和它的长期对话、项目约定、上下文记忆。通过 `sessions_spawn` 拉起的子 agent 拥有自己独立的 session 文件和上下文窗口，跑完后只把**最终回复**作为一条结果消息交回主会话。

这个机制天然就是隔离边界——但很多人用着用着，主会话还是炸了。

## 问题：污染的三个来源

1. **材料前置**：让子 agent 分析 20 个日志文件，但主 agent 在 spawn 之前就自己 `cat` 了一遍，中间输出全进了主上下文。
2. **结果复述**：子 agent 回了一份长报告，主 agent 又在回复里原样转述，双重放大。
3. **主动打通**：把主 session id 传给子 agent，或让子 agent 用 `sessions_send` 往主会话塞过程消息，隔离形同虚设。

## 做法

**第一步：spawn 提示词写清边界和输出预算。** 我的模板固定四段：

```text
任务：审计 workspaces/logs/ 下全部 *.log
范围：只读该目录，禁止访问 ~/.openclaw
输出：≤10 行结论 + 问题文件清单，写入 summary.md
禁止：向主会话发送过程消息
```

并明确告诉子 agent："你的最终回复会被主 agent 直接采用，不要输出过程流水。"

**第二步：传引用，不传内容。** 主 agent 只给路径、任务编号、上下文摘要，让子 agent 自己去读原材料。

**第三步：文件系统再隔一层。** 给子 agent 分配独立 workspace 子目录（如 `workspaces/<task>-<date>/`），产物落盘在子目录，主 agent 最后只读 `summary.md`。配置里开 sandbox 的 non-main 模式，让子会话跑在容器里：

```jsonc
// openclaw.json 片段
"agents": { "defaults": { "sandbox": { "mode": "non-main" } } }
```

**第四步：验证隔离生效。** 用 `openclaw sessions` 或 `sessions_list` 确认子会话独立存在；跑完后单独 reset 子会话，主会话 token 占用不变——这就是隔离生效的直接证据。

## 踩坑点

- **沙箱 ≠ 上下文隔离。** sandbox 隔离的是进程和文件系统；上下文隔离靠"子会话独立 + 只回传摘要"。两件事别混。
- **子 agent 里 `sessions_send` 回主会话**会绕过隔离，过程中间态又灌回去。过程沟通留在子会话内解决。
- **不写输出预算**，子 agent 默认给长报告，污染主会话。
- **并发子 agent 共用同一 workspace**，临时文件互相覆盖，按任务分目录。
- **cron / heartbeat 触发的重任务跑在主会话里**，等价于自我污染，改为 spawn 子 agent 或拆独立 agent。

## 可复用建议

- 一条核心原则：**主会话只留"决策与状态"，子会话吃"过程与体积"。**
- 把 spawn 提示词模板固化成团队约定，四段结构（目标/范围/输出预算/禁令）直接复用。
- 三类任务默认走子会话：重 IO、批量验证、探索性搜索；需要长期记忆的对话留在主会话。
- 定期检查 sessions 目录体积，主会话异常增长时优先排查"哪个任务没走子会话"。

## 总结

会话隔离不是一个开关，而是三层配合：**session 文件天然分离是底座，提示词约束回传体积是关键，sandbox/独立 workspace 是兜底**。做到"子 agent 只带回结论"，主会话的上下文窗口才能真正留给需要长期记忆的东西。下次遇到主会话莫名其妙变笨，先看看是不是有活儿本该派出去。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-21/efc9e4660a6bae00.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-21/77d21c1b78b70d89.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-21/881c6931b460dafd.png)

