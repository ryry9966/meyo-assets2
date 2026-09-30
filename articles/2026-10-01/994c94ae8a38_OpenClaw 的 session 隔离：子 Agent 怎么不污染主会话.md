---
title: OpenClaw 的 session 隔离：子 Agent 怎么不污染主会话
feedId: 39920
source: 综合讨论
publishedAt: 2026-10-01
---

## 背景

OpenClaw 的主会话本质是一份持续追加的 transcript：频道消息、工具调用、exec 输出、浏览器抓取结果，全部按顺序堆在同一个上下文里。日常问答没问题，但一旦让主 Agent 亲自跑重活——翻几十个网页做调研、批量改代码——中间产物就全留在 transcript 里了。

## 问题

典型的三类污染：

- **上下文膨胀**：token 消耗上涨，compaction 频繁触发；
- **注意力污染**：几轮之前无关的报错和日志，影响后面的回答质量；
- **任务串味**：A 任务的中间状态干扰 B 任务。

简单 `/new` 能清场，但会把当前任务还需要的上下文一起丢掉。

## 做法

OpenClaw 的解法是子 Agent：主会话通过 `sessions_spawn` 起一个独立 session，核心机制有四点：

1. **transcript 隔离**。子 Agent 有自己的 session key 和会话文件（`agents/<agentId>/sessions/` 下的 jsonl），它的几十次工具调用只写进自己的文件；主 transcript 里只会出现一条 spawn 调用和一条最终结果。
2. **返回值收敛**。子 Agent 结束时，主会话拿到的只有它的最终答复。把它当"返回字符串的函数"，不是"共享桌面的同事"。
3. **上下文显式传递**。子 Agent 默认看不到主会话历史。需要什么背景，写进 spawn 的 task prompt，或让它自己去读文件。
4. **产物落盘**。让重任务的输出写文件（报告、diff、数据），主会话只保留一句结论加路径。需要彻底清场时 `/new`，长期知识沉淀到 memory 文件，别让 transcript 当记忆。

一次典型流程：主会话收到"调研 X 并给建议"→ 主 Agent 调 spawn，prompt 写清目标、约束、输出格式、结果写入 `~/workspace/x-report.md` → 子 Agent 在自己 session 里跑完十几次工具调用 → 主会话收到 200 字结论加文件路径，上下文几乎没涨。

## 踩坑点

- **spawn prompt 里塞大段原文**：等于把污染提前打包。正确姿势是给文件路径，让子 Agent 自己读。
- **以为子 Agent"知道"刚才聊了什么**：它不知道。子 Agent 表现差，多数是缺上下文，不是模型不行。
- **不约束输出格式**：子 Agent 回来 2000 字散文，主会话照样被撑大。要求"不超过 N 字 + 结构化要点"。
- **递归 spawn**：子 Agent 再开子 Agent，成本和排障难度指数级。除非明确需要，禁止嵌套。
- **排障看错地方**：子 Agent 失败时去翻它自己的 session jsonl，主会话日志里只有失败结果。
- **session 隔离 ≠ 文件系统隔离**：两个子 Agent 并发写同一目录照样打架，执行环境隔离要配合 sandbox 或独立 workspace。

## 可复用建议

- 把子 Agent 任务当**纯函数**设计：输入 = 任务描述 + 文件引用，输出 = 短结论 + 产物路径。
- 固定一个 spawn 模板，五段式：**目标 / 边界 / 输出格式 / 产物位置 / 禁止事项**，比每次自由发挥稳定得多。
- 主会话保持"轻"：定期 `/new`，重活外包，结论写进 memory，不指望 transcript。
- 用 `/status` 观察各 session 的 token 占用，哪个异常膨胀就去查它的 jsonl。

## 总结

OpenClaw 的 session 隔离是 transcript 级隔离：中间过程留在子会话文件里，主会话只收结果。用好它的前提是把任务边界设计清楚——显式传上下文、约束输出、产物落盘。子 Agent 不是并行帮手，而是帮你把脏活和噪音挡在主会话外面的一道闸门。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-01/d895404de952ee15.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-01/a1bcabdb267f82ef.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-01/2f0f875e2b746b4b.png)

