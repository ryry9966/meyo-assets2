---
title: OpenClaw 的 session 隔离：子 Agent 怎么不污染主会话
feedId: 39881
source: 综合讨论
publishedAt: 2026-09-30
---

## 背景

OpenClaw 里每个 agent 默认一条主会话（main session）：消息历史、工具调用结果、compact 摘要都在同一条上下文里滚动。跑子 agent 的动机很直接——并行干重活、隔离长任务。但如果子 agent 的会话设计不对，等于把脏活的全部中间产物又倒回了主会话，隔离就成了摆设。

## 问题：污染长什么样

- **上下文膨胀**：子 agent 直接在主会话里跑长任务，几十轮工具调用的原文全部进入主上下文，一次抓取任务能吃掉几十 k token。
- **compact 提前触发**：历史被压掉，真正该长期保留的约定和偏好反而丢了。
- **文件串味**：子 agent 直接写 workspace 的共享文件，把主 agent 的笔记或配置覆盖。
- **排查困难**：翻主会话 transcript，分不清哪条输出是谁写的。

## 做法

1. **子 agent 用独立会话 spawn**。通过 `sessions_spawn` 拉起的子会话天然与主会话隔离，任务参数直接写进 spawn prompt，不要让子 agent 依赖主会话历史去"猜"任务。字段名随版本有差异，改配置前先对照当版 docs，或跑一次 `openclaw doctor`。
2. **约定返回契约**：子 agent 只回传"结论 + 关键数字 + 产物路径"，原始大段输出落盘到 workspace 文件，主 agent 需要细节时再按路径读取。
3. **收窄工具白名单**：子 agent 只给完成任务必需的工具，尤其是写类工具（文件写、消息发送）能不给就不给。
4. **大输出走文件，不走会话**：会话里只留指针。
5. **定期清理**：用 `openclaw sessions` 查看占用，idle 的子会话及时清掉，别让僵尸会话堆在后台。

## 踩坑点

- **污染换了个方向**：子 agent 回传时把"完整报告"整段贴回主会话，等于白隔离。返回契约要写死"只回摘要"。
- **共享 workspace 并发写**：两个子 agent 同时写同一文件。给每个子 agent 分配独立子目录，用完再合并。
- **会话隔离了，上下文来源没隔离**：子 agent 又去读主 agent 的记忆文件，读到过期约定照着执行。要么复制快照，要么明确禁读。
- **sandbox 路径映射**：容器内子 agent 回传的产物路径是容器内路径，主 agent 在宿主机读不到。回传前先归一化。
- **主会话 reset 后的引用失效**：子 agent 回传的路径指向已被清理的临时文件，交接产物一律放持久目录。

## 可复用建议

- 把"子 agent 只回摘要 + 文件路径"写进 `AGENTS.md`，让模型自己守规矩，比每次在 prompt 里强调可靠。
- 记住一条原则：**会话是易失的，文件是持久的**。重要产物一律落盘，会话只做过程。
- 把"主会话 token 增速"当作污染的告警指标，异常上涨先查最近 spawn 的子会话。
- 排障顺序固定：sessions 列表 → 子会话 transcript → workspace 文件时间戳，三步基本能定位。

## 总结

一句话概括：**主会话留给决策和对话，子 agent 留给干活，中间靠"摘要 + 文件路径"交接**。隔离不是开个开关就完事，返回契约、工具白名单、目录规划这三件套配齐，主会话才能长期保持干净。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-30/a2a34e0bf43ebbe0.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-30/0a645336ee72da70.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-30/b439fe6d28698c05.png)

