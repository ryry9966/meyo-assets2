---
title: 子 Agent 不污染主会话：OpenClaw 的 session 隔离实践
feedId: 42238
source: 综合讨论
publishedAt: 2026-10-22
---

### 背景

OpenClaw 里一个 agent 实例对应一条长期会话：频道消息、工具调用、工具输出全部追加进同一份 transcript。跑得越久，上下文越大，响应越慢。为了让主 agent 干活不带累赘，OpenClaw 提供了子 agent 机制（`sessions_spawn`，配合多 agent 配置），子 agent 拥有独立的 session 文件和独立的上下文窗口。

### 问题

隔离没做对时，污染通常从两个方向发生：

- **向内污染**：子 agent 的中间过程（长工具输出、报错重试、大段检索结果）被写进主会话 transcript，token 消耗骤增，主 agent 被过程细节带偏，重要历史被挤出上下文。
- **向外污染**：子 agent 直接调用消息工具往同一个频道发内容，用户看到两条"性格不一致"的回复，主会话的对话节奏被打断。

### 做法

1. **确认隔离边界**。每个 agent（含 spawn 出来的子 agent）都有独立的 sessions 目录（`~/.openclaw/agents/<agentId>/sessions/` 下的 jsonl）。子 agent 的工具调用只写自己的 transcript，父会话只接收最终一条结果摘要。spawn 前后用 `sessions_list` 核对，确认子会话真的另起了一条。
2. **把 spawn prompt 写成接口**。只给任务所需的事实、约束和期望的输出格式，不要把主会话历史整段贴进去——transcript 隔离挡不住你主动塞进去的上下文。
3. **给子 agent 独立身份和工作区**。在 `openclaw.json` 的 agents 配置里用单独的 agent id + workspace，避免共享 memory 文件造成间接污染。
4. **管住子 agent 的行为**。设置超时和最大轮数；确认 spawn 的结果回报行为（announce 相关参数按当前版本文档核对）；结果回来后由主 agent 消化、压缩，再以自己的口吻回复用户。

### 踩坑点

- **假隔离**：最常见。transcript 分开了，但 prompt 里塞了整段历史，上下文照样爆。
- **旁路输出**：子 agent 绕过父会话直接发频道消息，隔离形同虚设。给子 agent 收紧消息类工具权限。
- **共享 workspace**：文件系统不在隔离范围内。子 agent 改了主 agent 之后要读的文件，污染换个形式回来。
- **共享 memory**：子 agent 的记忆写入会影响主 agent 的后续会话，需要独立身份或关闭写入。
- **没有超时**：隔离只挡住了污染，挡不住烧 token。失控的子 agent 会安静地一直跑。

### 可复用建议

- spawn prompt 三段式：输入 / 约束 / 输出格式，当 API 契约来写。
- 一次 spawn 一个任务，不在子 agent 里继续派生孙 agent，除非确实需要树状并发。
- 用 label 命名子会话，排障时 `sessions_history` 一查一个准。
- 定期清理过期的子会话文件，`sessions/` 目录会悄悄变大。
- 子 agent 默认最小权限：能只读就只读。

### 总结

OpenClaw 的 session 隔离解决的是 transcript 层面的污染：子 agent 的过程记录不进父会话，父会话只拿结论。但文件系统、memory、频道输出这三条旁路要自己封住。隔离不是配置一开就完事，而是一个边界设计问题——想清楚父子之间交换什么、不交换什么，子 agent 才是助力而不是上下文黑洞。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-22/05675f045acdcd7d.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-22/64330254275579e4.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-22/0f668b8e3a2c0983.png)

