---
title: OpenClaw session 隔离实践：子 Agent 如何不污染主会话
feedId: 42293
source: 综合讨论
publishedAt: 2026-10-22
---

## 背景

OpenClaw 的主会话是和用户长期对话的唯一上下文：系统提示、记忆引用、全部历史都压在里面。子 Agent、定时任务、MCP 工具调用本应各开各的 session，互不打扰。但实际用下来，"隔离"并不是默认就干净的——配置和用法稍有不慎，子任务的中间产物就会回流到主会话。

## 问题：污染是怎么发生的

常见的三条回流路径：

1. **结果回灌过厚**：子 Agent 返回的不是结论，而是整段执行日志。主会话被塞进几千 token 的工具输出，之后每一轮对话都在为这些垃圾买单。
2. **共享写入口**：子 Agent 和主 Agent 指向同一个 workspace，甚至都能写 MEMORY.md。子任务跑完，主会话的记忆里混进了任务过程记录，久了会出现"人格漂移"。
3. **同 session 复用**：图省事直接在主会话里用 cron/heartbeat 跑长任务，执行痕迹全部留在主历史里，触发 compaction 后早期重要上下文反而被挤掉。

## 做法

我们的约定是把三条线分开：上下文线、文件线、写权限线。

**1. 上下文隔离。** 子 Agent 一律通过 agent 工具派生，拿到独立 session id，落在 `~/.openclaw/agents/<agent>/sessions/` 下，和主会话文件物理分开。派生时在任务 prompt 里写明："你是临时执行者，返回只给结论和关键数据，不贴过程"。回收时只取最终 deliverable。

**2. 文件隔离。** 文件密集型任务给子 Agent 单独的 workspaceDir，或至少一个 tmp 子目录，结束后由主会话只搬运结果文件。绝不让多个 agent 同时写同一个工作区。

**3. 写权限隔离。** 临时子 Agent 默认关掉 memory 写入，只有明确要沉淀结论的任务才显式放开。主会话的记忆只由主会话自己维护。

验证方法很朴素：跑完一个子任务后，看主会话 session 文件的增量行数和 token 消耗，暴涨说明回流没控住；再 grep 一下子任务的关键词有没有出现在主 session 的 JSONL 里。

## 踩坑点

- 子 Agent 并不继承主会话的完整系统提示，工具集也可能不同，别假设"它能像主 Agent 一样行事"。
- 长任务中途崩溃时，子 session 文件会残留在 sessions 目录里，记得定期清理，否则 session 列表越来越乱。
- MCP 工具的输出长度不受你控制，让子 Agent 先做摘要再返回，比直接透传省得多。
- 并发子 Agent 写同一个 git 工作区，会产生难以归因的脏状态，日志里看着还像是主 Agent 干的。

## 可复用建议

- 给子 Agent 固定一个模板 prompt：身份（临时执行者）+ 交付物格式 + 禁写项（memory、主工作区）。
- 把"主会话 token 增量"当回归指标：每次改子 Agent 配置后跑一次对照。
- cron / heartbeat 这类常驻任务用独立 session，跑完即弃，不在主历史留痕。
- 记忆写入当特权操作：默认拒绝，按任务白名单放开。

## 总结

session 隔离不是 OpenClaw 的某个开关，而是一套约定：上下文各自开、文件各写各的、记忆默认只读。子 Agent 的价值在于干脏活，主会话的价值在于保持干净和连续。把回流通道一条条堵住，长跑的主会话才不会越用越钝。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-22/4545467e58735949.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-22/88e652322a071c2d.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-22/b658b4f07a925cb2.png)

