---
title: OpenClaw session 隔离实战：子 Agent 干脏活，主会话只收结论
feedId: 40284
source: 综合讨论
publishedAt: 2026-10-03
---

# 背景

OpenClaw 的 Gateway 会为每个「agent × 通道」组合维护一份持久 session，对话历史、工具调用、工具输出都在里面持续累积。上下文一旦逼近阈值就触发 compaction，而 compaction 本质是有损压缩——丢掉的往往是你以为还在的细节，比如用户偏好、项目约定。

# 问题

主会话最该承载的是「与用户的连续对话 + 决策上下文」，但实际用起来经常被三类噪音污染：

1. **长工具输出**：读日志、跑测试、抓网页，一条命令几千 token；
2. **多步试错**：调试型任务中间失败十几次，全过程都留在主会话里；
3. **批量操作**：改 30 个文件，每个 diff 都要过一遍上下文。

结果是 compaction 频繁触发、响应变慢、成本上涨，真正重要的长期记忆反而被压没了。session 隔离就是冲这个来的。

# 做法与步骤

核心工具是 `sessions_spawn`：它启动一个子 agent run，拥有独立 session key 和独立上下文。子代理所有工具调用与输出都留在自己的 session 里，结束后只把一份结果摘要交回主会话。

**1. 配一个专职干活的 agent。** 在 `~/.openclaw/openclaw.json` 的 `agents` 里加一项，独立 workspace，模型可以选便宜的——机械性工作不需要最强模型：

```json
{
  "agents": [
    { "id": "main", "workspace": "~/openclaw-main" },
    { "id": "task", "workspace": "~/openclaw-task", "model": "<低成本模型>" }
  ]
}
```

**2. 在主 agent 的 AGENTS.md 里写死分流规则**，例如：「凡运行长命令、批量改文件、读大段日志或网页，必须用 `sessions_spawn` 派给 task，禁止在主会话直接执行。」

**3. 把 task 写成自包含 prompt**：目标、输入（一律绝对路径）、输出要求（结论 + 改动文件清单 + 失败原因）、超时。子代理看不到主会话历史，缺什么上下文就在 task 里补什么。

**4. 观察验证**：`openclaw sessions --active` 能看到 `agent:task:...` 的独立 session 在跑，`openclaw sessions history <key>` 可回看过程。跑完后主会话只多一条结构化结果。

# 踩坑点

- **相对路径失效**：子代理默认在自己的 workspace 里跑，主会话里说的「那个文件」到子代理那边就找不到。task 里一律用绝对路径。
- **结果摘要太虚**：不写明输出要求，子代理可能只回一句「完成了」，主 agent 还得自己重查一遍，隔离白做。
- **spawn ≠ send**：`sessions_send` 是往已有 session 写消息，用错等于把噪音直接灌回主会话；要隔离就用 spawn。
- **超时砍任务**：长任务记得给足 `timeoutSeconds`，否则子代理中途被杀，主会话只收到一个模糊失败。
- **并发派发要打 label**：同时起多个子代理，结果回来容易混，加上 `label` 方便在 `sessions_list` 里对账。

# 可复用建议

- 一条判断原则：**主会话只留决策和结论，过程性噪音全部下沉**。这个动作的中间输出你三天后还会看吗？不会，就 spawn。
- 把 task prompt 模板固化进 AGENTS.md，避免每次现编。
- 定期 `openclaw sessions --active` 巡检，清掉遗留的僵尸 session，别让隔离变成堆积。
- compaction 是兜底不是替代：前者被动丢信息，后者主动不产生噪音，别指望靠压缩解决问题。

# 总结

session 隔离本质是上下文预算管理：主会话的 token 最贵，应该花在对话质量和长期记忆上；脏活放进一次性子 session，用完即走。配置成本十分钟左右，换来主会话长期干净、compaction 明显变少。建议从「批量文件操作」和「日志排查」这两类最脏的任务开始试点，跑一周再决定推广范围。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-03/9886635406f09bfa.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-03/01fef5b602b2675c.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-03/06b6eb5cb9e43389.png)

