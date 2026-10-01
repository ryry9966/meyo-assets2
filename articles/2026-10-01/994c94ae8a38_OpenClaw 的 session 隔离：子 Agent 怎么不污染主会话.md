---
title: OpenClaw 的 session 隔离：子 Agent 怎么不污染主会话
feedId: 39976
source: 综合讨论
publishedAt: 2026-10-01
---

## 背景

OpenClaw 的主会话是长期资产：它承载你与 agent 的全部上下文、偏好和未完成任务。跑深度调研、批量处理文件、长链路工具调用时，如果直接在主会话里干，很快会把它撑爆——上下文膨胀、token 费用上涨、agent 开始"复述"中间过程。

OpenClaw 提供了子 agent 机制（以 `sessions_spawn` 为例）：把重活丢给一个拥有独立 session 的 agent，跑完把结果带回主会话。机制是现成的，但"隔离"不会自动生效——它取决于你怎么用。

## 问题：污染是怎么发生的

实际用下来，主会话被污染主要有三条路径：

1. **过程回灌**。子 agent 的中间结论、工具日志、报错重试，通过最终回复整段带回主会话，一条任务下来几千 token。
2. **session 复用**。子 agent 没拿独立 session key，或两次 spawn 共用同一个 key，上一次的残留上下文渗进下一次。
3. **反向污染**。为了"给足上下文"，把主会话历史整份塞给子 agent，子 agent 的输出又带着旧话题回来，两条线搅在一起。

## 做法

**第一步：spawn 时给独立 session。** 每个子 agent 使用独立 session key（或依赖默认的每次新建行为），不要手工复用。跑完可以到 `~/.openclaw/agents/<agentId>/sessions/` 下确认确实多了一个独立 session 文件，而不是并入了主会话。

**第二步：定义返回契约。** 这是最关键的一步，在任务 prompt 里明确：

- 只返回最终结论，用结构化格式（JSON 或要点式 markdown）；
- 过程日志、原始抓取内容一律不回传；
- 大产出写文件，回复里只给路径 + 200 字以内摘要。

一个可以直接抄的模板句：「将结果写入 `/tmp/research/out.md`，回复中只包含：结论 3 条、文件路径、遇到的可复用错误模式。」

**第三步：收窄工具与目录。** 子 agent 只挂它需要的工具（比如只给 browser 或只给 exec），工作目录指向独立的 scratch 目录。注意文件系统默认是共享的，目录隔离要自己做。

**第四步：验证。** 跑一个典型任务，前后对比主会话的 token 用量和消息条数。健康的隔离应该是：主会话只多一条工具结果（几十到几百 token），子 session 里躺着完整过程。

## 踩坑点

- **最终回复超长**。子 agent 天然倾向写完整报告，不约束的话照样把主会话撑大。字数上限要写进任务 prompt，不能靠默契。
- **并发写同一文件**。两个子 agent 共享文件系统，同时写一个输出文件会互相覆盖。按任务分配独立子目录。
- **主 agent 抢着复述**。收到结果后主 agent 有时自动展开一大段总结。如果不需要，在系统提示里加一句"子任务结果未经要求不要展开"。
- **超时与取消**。被 kill 的子 agent 会留下半截 session，一般无害，但定期清理旧 session 能省磁盘，也避免误读。

## 可复用建议

- 心智模型一句话：**主会话存决策和状态，子 session 存过程**。
- 把"子 agent 任务 prompt 模板"（含返回契约）存成固定片段，每次 spawn 复用，别现场手写。
- 每周花一分钟过一遍 session 清单，看有没有异常膨胀的主会话或孤儿 session。
- 给子 agent 的上下文按需裁剪：只传任务相关的事实，不传聊天历史。

## 总结

session 隔离在 OpenClaw 里不是一个开关，而是一组工程习惯：独立 session key、明确的返回契约、收窄的工具与目录、跑完即验证。做到这四点，主会话可以长期保持干净——子 agent 随便开、随便扔，重活干得再脏，也不影响主线的判断力。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-01/4fd88b29ee929b19.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-01/2d96573e9c9b062b.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-01/5513012ab15ac235.png)

