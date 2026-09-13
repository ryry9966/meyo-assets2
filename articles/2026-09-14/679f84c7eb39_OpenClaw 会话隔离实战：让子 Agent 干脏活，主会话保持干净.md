---
title: OpenClaw 会话隔离实战：让子 Agent 干脏活，主会话保持干净
feedId: 37434
source: 综合讨论
publishedAt: 2026-09-14
---

## 背景

OpenClaw 里每个对话对应一个 session，主 session 里堆着系统提示、对话历史、工具调用记录和消息队列。上下文窗口是有限的，用得越久越长，长到一定程度就触发 compaction，早期细节被压缩甚至丢掉。这是所有长任务不稳定的根源之一。

## 问题

最容易翻车的场景：让主 Agent 直接扛重活——批量读文件、跑长命令、多步排查。几十轮工具调用的输入输出全部留在主 session，后果是：

- 上下文被中间过程塞满，真正重要的对话被挤出窗口；
- compaction 之后 Agent“失忆”，忘了之前的约定；
- 用户端看到满屏执行细节。

本质上，这是“执行上下文”和“对话上下文”没有分开。

## 做法

OpenClaw 的解法是子 Agent session 隔离，核心是 `sessions_spawn`：

1. 主 Agent 判断任务偏重，调用 `sessions_spawn`，传入一段**自包含**的任务描述；
2. 系统开一个全新子 session（ID 形如 `agent:main:subagent:xxx`），子 Agent 只看到这段 task prompt，**不带主会话历史**；
3. 子 Agent 在自己的上下文里跑完整工具循环——读文件、执行命令、试错，全部留在子 session；
4. 结束时只有最终摘要回传给主 Agent，中间过程不进主上下文；
5. 需要审计时，用 `sessions_list` / `sessions_history` 查子 session 完整轨迹。

一个关键点：子 Agent 和主 Agent 共享 workspace 文件系统，所以“传数据”优先靠文件——子 Agent 把产物落盘，主 Agent 之后直接读，比往回传消息里塞大段内容稳得多。

## 踩坑点

1. **task prompt 不自包含**。子 Agent 看不到主会话，“按上面说的做”在子 session 里等于空指针。目标、约束、路径、验收标准必须全写进 spawn 的 prompt。
2. **期待子 Agent 直接回频道**。子 Agent 默认不向频道发消息，结果只能经主 Agent 转述；确需跨 session 通信要走 `sessions_send`，别默认它有这个行为。
3. **并发拉太多子 Agent 不清理**。子 session 会留存，`sessions_list` 一查一屏，排查时全是噪音。跑完记得删或确认状态。
4. **把“共享文件”当成“共享上下文”**。文件系统共享，但对话和记忆不共享，子 Agent 不知道你的偏好，除非写进 prompt。
5. **主 Agent 该隔离时不隔离**。明明是十步以上的工具循环，它却选择自己干——通常是系统提示里没写清“什么情况必须 spawn”。

## 可复用建议

- 定一条分界线：预计超过 5–10 轮工具调用、或会产生大段输出的任务，一律 spawn；
- task prompt 模板化：目标 / 输入路径 / 约束 / 期望返回格式（如 JSON 摘要 + 产物文件路径）；
- 要求子 Agent 回传**结构化摘要**，控制体积，细节落盘；
- 主 session 只留对话、决策、结论——它越干净，compaction 越晚发生；
- 定期 `sessions_list` 清理僵尸子 session。

## 总结

session 隔离的价值不在“多开几个 Agent”，而在上下文分层：主会话是稀缺资源，只放对话和决策；执行过程下沉到子 session，用文件交换数据，用摘要回传结论。把这个习惯建立起来后，长任务的稳定性会有肉眼可见的改善。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-14/dd46e65f99ba0df7.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-14/e765fffe2e014dbc.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-14/fbe3829afe2472d8.png)

