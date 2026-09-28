---
title: OpenClaw session 隔离实战：让子 Agent 干脏活，主会话保持干净
feedId: 39344
source: 综合讨论
publishedAt: 2026-09-28
---

# OpenClaw session 隔离实战：让子 Agent 干脏活，主会话保持干净

## 背景

OpenClaw 里每个聊天通道对应一条主 session，你和 agent 的全部对话、每次工具调用的输出，都堆在同一份上下文里。上下文窗口是有限资源，逼近上限就会触发 compaction，旧对话被压成摘要——而压缩是最容易丢细节的环节。接了 MCP 之后更明显：外部工具一次查询返回多少 token，你控制不了。

## 问题

最典型的翻车现场：让主 agent「把这 30 个日志文件都读一遍找报错规律」。几十 K token 的工具输出直接灌进主会话，三个后果：

1. 触发 compaction，前面聊定的技术方案细节被压没了；
2. 后续每轮请求都背着这坨中间产物，费用和延迟一起涨；
3. 长任务同步阻塞，这期间你在同一会话里干不了别的。

## 做法

1. **重活一律 `sessions_spawn`**。子 agent 拿到独立 session、独立上下文窗口，主会话只接收它的最终结论。
2. **任务 prompt 必须自包含**。子 agent 看不到主会话历史，文件路径、验收标准、输出格式要一次写全，别写「刚才说的那个文件」。
3. **限定回传体积**。任务里明确要求：只回 300 字内结论 + 产物文件路径。否则子 agent 干完把全文搬回来，等于换个地方污染。
4. **进度走旁路**。中途状态用 `sessions_send` 推到 thread，不进主线；排查用 `sessions_list` / `sessions_history` 看子会话实际发生了什么。
5. **收敛工具面**。给子 agent 配独立 tool profile，比如只给 exec 和 browser，不给 message——防止它绕过编排逻辑直接跟用户说话。
6. **规则写进 AGENTS.md**。固化一条：预期工具输出超过几千 token 或耗时超过一分钟的任务必须 spawn，主会话只做编排和决策。

## 踩坑点

- 并发 spawn 五个以上容易撞 rate limit，而且每个子 agent 会重复读同一批文件，成本翻倍。同类任务合并成一个大任务更划算。
- session 落盘在 `~/.openclaw/agents/<agentId>/sessions/`，JSONL 直接可读。怀疑「上下文被污染」时别靠回忆猜，翻文件。
- 子 agent 里再 spawn 子 agent 有嵌套深度和循环风险，层层转包最后没人对结果负责。
- 旧 session 文件会一直堆积，注意清理周期，尤其是高频跑 cron 任务的 agent。

## 可复用建议

AGENTS.md 里可以直接放这段：

```markdown
- 批量读文件、长耗时搜索、网页抓取：一律用 sessions_spawn
- spawn 的任务 prompt 必须自包含，回传限制在 300 字内
- 子 agent 不拥有 message 工具
- 主会话保留决策权和最终落笔
```

判断标准一句话：**这个任务产生的中间输出，你愿不愿意让它永久留在主会话里？不愿意，就隔离。**

## 总结

session 隔离的本质不是「多开几个 agent」，而是给上下文做预算分配：主会话的每一个 token 都该花在决策和对话上，中间产物留在子会话里烧完即弃。OpenClaw 的 sub-agent 机制把这件事做得足够轻，剩下的只是约束好你的 agent 别偷懒走捷径——把规则提前写下来，永远比事后翻 JSONL 排查便宜。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-28/a45aa44fc76ce8bf.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-28/383d88182b91092c.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-28/70b8a153980b53b8.png)

