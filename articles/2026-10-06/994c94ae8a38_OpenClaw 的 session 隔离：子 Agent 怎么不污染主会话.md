---
title: OpenClaw 的 session 隔离：子 Agent 怎么不污染主会话
feedId: 40624
source: 综合讨论
publishedAt: 2026-10-06
---

## 背景

OpenClaw 的主会话是个长跑选手：接在 Telegram/Discord 等渠道上，历史一直滚动，靠 compaction 续命。日常对话、临时指令、工具调用结果全都挤在这一个窗口里。当你开始让它跑"调研某个库""批量审查十个 PR""整理一个目录"这类子任务时，麻烦就来了——这些任务的中间过程（搜索结果、文件内容、报错栈）体积远大于最终结论。

## 问题

如果子任务直接在主会话里展开，会有三个后果：

- 中间产物把上下文提前撑满，compaction 频繁触发，重要的长期历史被压掉；
- 子任务的过程性指令与主会话的长期规则互相串扰；
- 敏感信息被复制进更多轮次，扩散面变大。

所以思路不是"让子 agent 更聪明"，而是让它换个房间干活：`sessions_spawn` 会开一个完全独立的 session，子 agent 看不到主会话历史，只拿到你给的任务描述；干完通过 `sessions_send` 把结果送回来，这个 session 就可以废弃了。

## 做法

1. 在 AGENTS.md 里约定 spawn 触发条件：凡是"过程长、结论短"的任务（调研、批量操作、排障）一律 spawn，不占主会话。
2. 写任务描述模板，四个字段：目标、边界（允许碰什么、禁止什么）、输出格式（建议 JSON 或不超过 10 行的摘要）、终止条件。
3. 传最小上下文。不要贴主会话历史，只给结论所需的片段：相关文件路径、一行背景、当前状态。
4. 回传即焚。主会话只接收结构化摘要；想看过程时用 `sessions_history` 去子会话里翻，而不是把过程拉回来。
5. 收尾清理。确认结果后用 `sessions_list` 巡检，删掉废弃 session，避免孤儿文件堆积。

## 踩坑点

- **最常见的是假隔离**：spawn 时把整段对话历史塞进任务描述，窗口照样被吃满，只是换了个地方污染。任务描述控制在几百字内。
- 子 agent 自己再 spawn，没有约束就会递归。在任务边界里写死"不允许继续派生"。
- 回传结果太大：子 agent 把 50 行日志原样送回，主会话照样爆。模板里强制"只回结论与异常"。
- `/new` 只重置主会话，spawn 出来的 session 还留在 `~/.openclaw/agents/<id>/sessions/` 里，磁盘会被慢慢吃掉。
- 多个子 agent 并发写同一个 workspace 文件没有锁，要么分路径，要么只让一个写。
- cron/heartbeat 如果和子任务共用 session，定时任务的上下文会和子任务搅在一起，建议分开。

## 可复用建议

- 把"何时 spawn、任务模板、回传格式"写进 AGENTS.md，用规则代替每次临时决定。
- 主会话原则：只留决策和结论，过程性内容一律外置。
- 每周跑一次 `sessions_list` 做清理，当成维护例行动作。
- 复盘子任务用 `sessions_history`，别用"让它复述一遍"的方式把过程拉回主会话。

## 总结

session 隔离的本质是上下文预算管理：主会话是稀缺资源，只装结论、决策和长期规则；过程、噪声、中间产物全部下沉到子会话。OpenClaw 已经把机制（spawn / send / history / list）给全了，剩下的工程活是定好模板和纪律，并且真的执行。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-06/fe4967a76644a831.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-06/d1af0ff8826289de.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-06/56e1dbac28f974f3.png)

