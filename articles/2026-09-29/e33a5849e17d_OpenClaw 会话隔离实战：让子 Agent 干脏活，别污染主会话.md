---
title: OpenClaw 会话隔离实战：让子 Agent 干脏活，别污染主会话
feedId: 39564
source: 综合讨论
publishedAt: 2026-09-29
---

## 背景

OpenClaw 的主会话跟着渠道走——你私聊 bot 的那个对话，生命周期很长，历史会一直累积。默认情况下，所有工具调用的输出都进入当前 session 的上下文。跑得越久，context 越满，主 agent 就越"健忘"，费用和延迟也跟着涨。

## 问题：主会话是怎么被污染的

三个典型场景：

1. 让主 agent 直接读大文件、抓网页、调一个输出很大的 MCP 工具，几万 token 的原始数据瞬间灌进主会话，前面聊的内容实际等于作废。
2. 自动化任务（cron、heartbeat）和日常对话共用 session，夜里跑批的中间输出白天还在挤占上下文。
3. 多任务交错，A 任务的调试信息混进 B 任务的判断里，模型开始串台。

## 做法

**1. 重 IO 任务一律下沉到子 agent。** 用 `sessions_spawn` 开一个独立 session，spawn prompt 里写死三件事：目标、边界（不许做什么）、返回格式（比如"只回 300 字以内的结论 + 关键数据，JSON 格式"）。子 agent 中间折腾出的所有大输出都留在它自己的 session 里，主会话只收到摘要。

**2. 主会话只走决策路径。** 拆解任务、派发、汇总、回复用户——凡是"可能产生大段输出"的动作，条件反射式下沉。主会话里应该只看得到干净的结论，看不到过程。

**3. 工具侧兜底。** MCP server 的输出做分页或截断；不确定数据规模时，先让子 agent count 再决定拉多少，别一把梭。

**4. 长期记忆放文件，不放对话。** 用户偏好、项目约定写进 AGENTS.md 和 workspace 文件，而不是依赖对话历史。这样定期 reset / compact 主会话不丢关键信息。

**5. 临时 session 用完即弃。** 一次性任务的子 session 跑完就清理，别复用带着旧上下文去跑新任务。

## 踩坑点

- **子 agent 返回不设限等于白隔离。** 我见过子 agent 拎着一整段日志回来，主会话照样被撑爆。返回格式必须写进 spawn prompt，最好强制 JSON + 字数上限。
- **子 agent 共享 workspace 文件系统。** 两个子 agent 同时写同一个状态文件会互相覆盖，约定各自只写自己的子目录。
- **并发 spawn 别开太大。** 限流、费用先不说，主 agent 等待汇总时要挂起的结果也变多。实践里 2–3 个并发基本够用。
- **隔离不等于零成本。** 主会话的 system prompt + 各插件的工具描述本身就常驻上下文，插件装一堆同样是一种"污染"，用不上的就关。
- **子 agent 超时/失败要有兜底。** 主 agent 拿到空结果会开始瞎编，宁可让它显式报错、重试或换路。

## 可复用建议

- 建一个四段式子 agent prompt 模板：目标 / 禁止事项 / 输出格式 / 失败时行为。每次 spawn 只填空。
- 在 AGENTS.md 里写死规则："读大文件、抓网页、批量处理必须走子 agent 并返回摘要"，主 agent 会自动遵守，不用每次口述。
- 大输出工具统一包一层"先截断再返回"的 wrapper，在工具层解决问题，比在 prompt 里求模型自觉靠谱。

## 总结

Session 隔离的本质是控制信息流向：脏数据留在子 session，主会话只保留决策和摘要。OpenClaw 的 `sessions_spawn` 已经把机制给全了，剩下的功夫在纪律——模板化、限制并发、及时清理，都是一次配置、长期受益的事。主会话干净了，bot 才不会越聊越"失忆"。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-29/0386f53a26b01612.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-29/8d991c9e69dd9df9.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-29/902cd537293f7cd3.png)

