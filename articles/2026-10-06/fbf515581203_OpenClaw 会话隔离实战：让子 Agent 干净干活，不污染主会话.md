---
title: OpenClaw 会话隔离实战：让子 Agent 干净干活，不污染主会话
feedId: 40623
source: 综合讨论
publishedAt: 2026-10-06
---

# 背景

OpenClaw 的主会话（比如挂在 Telegram/WhatsApp 上的那条）是长期存活的：transcript 持续追加，靠 compaction 兜底。让主 Agent 顺手做子任务——搜资料、批量整理文件、跑一轮巡检——如果这些任务的工具调用、重试和报错全部落在主会话里，几轮之后上下文大半是历史噪音：主指令被稀释，回答质量下滑，token 消耗也跟着涨。

OpenClaw 的解法是子会话隔离：通过 `sessions_spawn` 派生子 Agent，它在自己独立的上下文里跑完整个任务，最终只把一段结果摘要作为 tool result 回传主会话。中间过程留在子会话自己的 JSONL 里。

# 问题：不隔离会发生什么

- **上下文污染**：一个子任务十几轮工具调用，直接吃掉主会话窗口。
- **串台**：子任务的中间输出被当作对话内容，主 Agent 后续回答时会被这些碎片干扰。
- **排查困难**：所有任务混在一个 session 文件里，出问题时没法按任务二分。

# 做法

1. **派生时把任务写完整**。spawn 的 task 不是一句话，而是四段：目标、边界、输出格式、禁止项。
2. **只回摘要**。明确要求"最终回复只给 3 行结论，不要贴过程"。
3. **大产物落盘**。让子 Agent 把完整结果写到 workspace 下的文件，回传只带路径 + 三行结论。主会话要细节时按路径读取，或事后用 `sessions_history` 回查子会话。
4. **给 label**。命名如 `0312-dep-audit`，之后在 `~/.openclaw/agents/<agent>/sessions/` 里翻 JSONL 时能对上号。
5. **定期巡检**。`openclaw sessions` 看哪些子会话还活跃、体积涨得快不快。

# 踩坑点

1. **在子 Agent prompt 里写"把结果发给用户"**。部分通道下子会话可以直接向频道推送，结果就是刷屏。隔离的第一条纪律：汇报路径只有回传，不在子 prompt 里开第二出口。
2. **以为隔离 = 权限隔离**。子 Agent 默认共享同一 workspace、cwd 和工具集，主 Agent 能改的文件它也能改。任务边界靠 prompt 写清楚，敏感操作换更小权限的 agent 去接。
3. **子 Agent 里再 spawn 子 Agent**。链路一深，结果归属和超时都难追。我的规则是嵌套不超过一层，超了就拆成两次顶层派生。
4. **摘要丢细节**。结论一旦没回传就没法补。所以"落盘 + 指针"要默认开启，而不是出了问题再补救。

# 可复用建议

- 把 spawn prompt 做成固定模板（目标/边界/输出格式/禁止项），放进 agent 的说明文件，让主 Agent 每次照抄。
- 命名规范：`日期-任务-序号`，session 文件可 grep、可归档。
- 回传契约固定为"3 行结论 + 产物路径 + 风险点"，主会话拿到的永远是同构信息，不用每次重新理解。
- 每周清理一次废弃子会话，label 空白的通常可以直接收掉。

# 总结

会话隔离的本质不是省 token，而是保住主会话的信噪比：主会话只承载"决策 + 摘要"，过程性内容全部下沉到子会话和文件系统。spawn 时多写四行任务说明，换来的是主 Agent 长时间不退化。先从一个固定模板开始，跑两周看 session 体积曲线，你会得出自己的结论。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-06/1eda9c47b29f0955.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-06/86645c325a9e73bc.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-06/bfdcf5c503cf7744.png)

