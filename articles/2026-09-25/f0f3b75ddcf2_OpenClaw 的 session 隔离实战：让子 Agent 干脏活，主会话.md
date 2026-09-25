---
title: OpenClaw 的 session 隔离实战：让子 Agent 干脏活，主会话保持干净
feedId: 38894
source: 综合讨论
publishedAt: 2026-09-25
---

## 背景

OpenClaw 的主会话是一条持续累积的上下文流：你和 agent 的对话、每次工具调用的结果、读进来的文件和网页内容，全部追加在同一条 transcript 里。直接后果是——一次重量级任务（通读一个仓库、调研十个网页）之后，主会话膨胀几万 token，之后每轮对话都背着包袱跑，compaction 变得频繁，早期的系统指令和用户偏好被压缩稀释，agent 行为开始漂移。

## 问题

重点不是“能不能干重活”，而是“干完重活之后主会话还能不能用”。我们想要的效果是：重活有人干，脏东西留在工人自己的房间里，主会话只收一份干净的工单回执。

## 做法

以“让 agent 调研一批 MCP server 并给出选型建议”为例：

1. **声明子 agent**。在 `openclaw.json` 的 `agents.list` 里加一项，比如叫 `research`：单独指定模型、收窄 tools 白名单（只给搜索和抓取类工具）、分配独立的 workspace 子目录。
2. **派活，且 prompt 自包含**。主 agent 通过 `sessions_spawn` 下发任务。关键是 spawn prompt 必须独立成立：目标、最小必要上下文、约束（不要继续派生子 agent、不要动 workspace 之外的文件）、输出格式（完整产物写到 `workspace/output/`，只返回 20 行以内的结论）。
3. **隔离生效的位置**。子 agent 拿到的是一个全新的 session key，从零开始跑。跑完后主会话里只落两条记录：一条 spawn 调用、一段返回摘要。中间几十次工具调用的 transcript 全部留在子会话里，随任务结束丢弃。
4. **按需审计**。要查细节时用 `sessions list` 看活跃会话，用 `sessions history` 回放某个子 agent 的具体操作——按需拉取，而不是默认全量灌进主上下文。

## 踩坑点

- **spawn prompt 里复制了一大段主会话历史**，想让子 agent“了解前情”。隔离等于白做，包袱原样转移。正确姿势是只传最小上下文。
- **子 agent 默认可以再 spawn 子 agent**。任务 prompt 里不写“不要继续派生”，遇到发散型任务可能嵌套出一堆会话。
- **session 隔离 ≠ 状态隔离**。子 agent 共享同一套文件系统，两个子 agent 并发写同一个文件会互相覆盖。要么分 workspace，要么约定各自的子目录。
- **返回摘要不设上限**，子 agent 忠实贴回三万字，主会话照样被污染。上限写进 prompt，并强制“详情落盘、只回结论”。
- **tools 白名单忘了收窄**。隔离的是上下文，不是权力——子 agent 默认可能拿到发消息、改配置的能力，按任务实际需要给。

## 可复用建议

- **经验法则**：预计要读 10 个以上文件、或工具循环超过几轮的任务，一律走子 agent。
- **固化一个 spawn prompt 模板**：目标 / 最小上下文 / 约束 / 输出格式 / 产物目录，五个字段填空即可。
- **心智模型**：主会话是控制面（control plane），只做决策和调度；子 agent 是数据面（data plane），负责吞吐和脏活。
- **定期清理** `~/.openclaw/agents/` 下历史 subagent session 文件，磁盘和审计都轻松。

## 总结

session 隔离的价值不在“多跑几个 agent”，而在控制主会话的信噪比：让主会话始终保持“指令清晰、历史干净”，把吞吐量外包给随时可丢弃的子会话。配置成本十分钟左右，换来的是长期稳定的 agent 行为——这笔账在 OpenClaw 上非常划算。有更复杂的隔离需求（比如多子 agent 并发写同一 workspace）欢迎在评论区讨论。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-25/20111017ac27aed8.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-25/48e5868b0fd47619.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-25/e1419562f0cd70b8.png)

