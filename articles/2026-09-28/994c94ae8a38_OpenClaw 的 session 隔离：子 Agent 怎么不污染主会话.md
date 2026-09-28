---
title: OpenClaw 的 session 隔离：子 Agent 怎么不污染主会话
feedId: 39216
source: 综合讨论
publishedAt: 2026-09-28
---

## 背景

OpenClaw 的主会话是一条不断增长的 context：系统提示、记忆注入、聊天历史、每一次工具调用与返回，全堆在一起。日常对话没问题，但只要让主 Agent 内联跑重活——批量读文件、连环调 MCP 工具、抓一堆网页——中间产物就会把 context 撑爆。compaction 一触发，早期定下的约束和细节就被压没了。

## 问题

三个典型症状：

1. **越聊越失忆**：长任务跑完，之前定的输出规范、路径约定在压缩后丢了。
2. **串行阻塞**：主会话一次只能推进一件事，一个慢工具把整段对话卡住。
3. **试错也占历史**：Agent 重试十次的工具输出全部留在主上下文里，而且是长期性的。

## 做法

核心机制是 `sessions_spawn`：把重活外包给子 Agent 的独立 session。子 session 的 key 带 `subagent` 段，context 是全新的——只有你传给它的任务书，没有主会话历史。跑完后，主会话只回收它的最后一条消息作为结果，中间所有工具噪音都留在子 session 里。

落地步骤：

1. **筛选该外包的活**：读多写少的调研、批量文件操作、会产生大量工具输出的 MCP 调用链，都适合 spawn。
2. **写自包含任务书**：子 Agent 不继承主会话记忆，目标、约束（时区、目录、输出格式）、交付物、超时都要显式写进去。
3. **限权**：通过 tools policy 给子 Agent 加 deny，禁掉消息发送和再 spawn，防止它直接回用户消息或递归开 Agent。
4. **隔离文件系统**：并行子 Agent 共享 workspace 会互相覆盖，给每个任务单独 scratch 目录。
5. **收割后清理**：结果拿到手就删子 session，别让列表里堆僵尸。

任务书模板（可直接抄）：

```
目标：…
约束：时区 UTC+8；只读写 /workspace/scratch/<task>/；输出中文。
交付：最后一条消息必须是结构化摘要——结论、关键数据、产出文件路径。
超时：runTimeoutSeconds 设 600，宁可失败重试也不要挂死。
禁止：给用户发消息、再 spawn 子 Agent。
```

## 踩坑点

- **忘写约束**：子 Agent 默认不知道你的时区和代码风格，结果格式全靠猜。
- **没加 deny**：工具策略默认继承，子 Agent 照样能发消息、能再开 Agent。
- **结果太"薄"或太"厚"**：任务书没要求结尾输出结构化摘要，关键信息埋在中间；要求太松，回传一大段照样污染主会话。
- **没设超时**：长任务挂着，占资源也占 session。
- **复盘别在主会话里猜**：用 `openclaw sessions` 看活跃列表，用 sessions history 类命令翻子 session 的完整过程，排查"它到底干了什么"快得多。

## 可复用建议

- 一条经验法则：**会产生大量工具输出的活 → 子 Agent；需要对话上下文的活 → 主会话**。
- 心智模型：主会话是控制平面，子 Agent 是工人，控制平面保持精简。
- 把任务书做成项目里的模板文件，spawn 时填空，减少遗漏。
- 定期 `openclaw sessions` 审计，清理长期不活跃的子 session。

## 总结

Session 隔离本质是 context 经济学：把中间产物隔离在子 session，主会话只留决策所需的信息。写好任务书、限好权、设好超时、收完即清——四个动作做到位，主会话就能长期保持"脑子清醒"。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-28/e2567c6b286fabe4.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-28/5ff1eafd677844b0.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-28/df21f7738198f1a2.png)

