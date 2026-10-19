---
title: OpenClaw 的 session 隔离：子 Agent 怎么不污染主会话
feedId: 42045
source: 综合讨论
publishedAt: 2026-10-20
---

## 背景

OpenClaw 的主会话是长上下文：你聊过的每一句、每次工具调用的返回、读过的文件片段，都会追加进 transcript（`~/.openclaw/agents/<agentId>/sessions/<sessionId>.jsonl`）。日常轻量使用没问题，但一旦主 Agent 亲自去跑重活——批量读仓库、跑长任务、并行调研——所有中间过程全部灌进主会话，上下文迅速膨胀，后面的对话质量明显下滑，token 成本也跟着涨。

OpenClaw 的子 Agent 机制（`sessions_spawn`）就是为这个场景设计的，核心思路一句话：**隔离执行，只回结论**。

## 问题

- **上下文污染**：中间 tool 输出直接写进主会话，模型注意力被无关过程淹没；
- **状态串扰**：多个任务挤在同一个会话里，并发时互相踩；
- **成本失控**：每次主会话请求都携带全部历史，重任务做一次，之后每次对话都在为它付费。

## 隔离是怎么做的

1. **每个 Agent 独立 session 文件**。子 Agent 有自己的 sessionId 和独立 transcript，中间步骤只落在自己的文件里，不向主会话追加。
2. **用 `sessions_spawn` 派生**。在 task 里写清目标，可选配工具白名单和模型档位，子 Agent 拿到干净的上下文窗口开跑。
3. **只回流一条结果**。子 Agent 跑完后，主会话收到的只是一条最终 result，几百个 read 结果不会跟着回来。
4. **task 必须自包含**。子 Agent 看不到主会话历史，项目背景、文件路径、验收标准都要显式写进 task。
5. **大产物落盘**。让子 Agent 把报告、代码写进 workspace 文件，返回摘要 + 文件路径，别让它把全文塞进返回值。
6. **定期清理**。用 `sessions_cleanup` 清过期子会话；注意主会话 `/new` 或 reset 并不会顺带删子会话。

## 踩坑点

1. **session 隔离 ≠ workspace 隔离**。默认共享 workspace，并发子任务同时写同一批文件照样冲突。重 IO 任务给独立工作目录，或者明确分工。
2. **返回值过大照样污染**。子 Agent 一开心把 3000 字结果全贴回来，等于换了个地方塞主会话。约定返回格式：结论 + 路径，控制在几百 token 内。
3. **没传上下文**。子 Agent 报“找不到文件”“不知道项目结构”，十有八九是 task 没写全，不是隔离机制的锅。
4. **嵌套派生**。子 Agent 再 spawn 子 Agent 有深度限制，别设计成递归派生的任务图，跑挂了不好排查。
5. **cron / webhook 触发的运行也是独立 session**。别误以为“同一个聊天窗口就共享记忆”，定时任务拿到的是干净上下文。

## 可复用建议

- **一任务一子 Agent 一 session**，task 自包含，当成一次函数调用来看待；
- **固定返回契约**：一句话结论 + 产物路径，超了就让子 Agent 自己压缩；
- **按任务配资源**：调研类给只读工具白名单和便宜的模型档位，写码类才放开写权限；
- **把派生策略写进 AGENTS.md**：明确“哪些事必须 spawn、返回格式是什么”，让主 Agent 形成稳定习惯，而不是每次即兴发挥。

## 总结

session 隔离的本质是控制信息流：中间过程留在子会话里消化掉，只有结论流回主会话。把派生时机、task 写法、返回契约这三件事固化成团队惯例，主会话就能长期保持干净，重任务再多也不会把上下文拖垮。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-20/d321403e2712a59f.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-20/c2668e28eeb73d78.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-20/3b16cbfd24930b83.png)

