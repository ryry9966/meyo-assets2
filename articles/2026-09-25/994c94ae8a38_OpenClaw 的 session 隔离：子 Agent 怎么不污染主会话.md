---
title: OpenClaw 的 session 隔离：子 Agent 怎么不污染主会话
feedId: 38981
source: 综合讨论
publishedAt: 2026-09-25
---

## 背景

OpenClaw 的会话模型很朴素：一个 agent + 一个通道用户 = 一个 session。主会话承载你的日常对话、决策和长期上下文，它的 context window 是最稀缺的资源——一旦被大量中间过程撑满，轻则触发 compaction 丢细节，重则整个 agent 表现劣化。所以做自动化流水线的人几乎都会用到子 Agent：把重活、脏活、多步骤探索外包出去。

## 问题

最直觉的做法很诱人：直接在主会话里让模型"自己查、自己跑"，或者随手用 sessions_send 把消息丢到别的 session。结果就是——几十次工具调用的输出全部堆在主上下文里，一次网页抓取之后主会话躺着十几页无关内容，下一轮对话开始答非所问。

更隐蔽的是反向污染：子 Agent 往主会话的工作区写文件、改状态，主会话后续步骤读到的是半成品。这两类污染叠加，基本就是你"agent 用着用着变笨"的主因。

## 做法

核心原则一句话：**子 Agent 是一次性 worker，进出都要收口。**

1. **用 sessions_spawn，不要用 sessions_send 顶替。** spawn 会创建一个独立 session key 的子会话，有自己的 context window，中间过程不写回主会话。短任务用前台模式阻塞等结果；长任务用后台模式，完成时以系统消息形式把结果回投。
2. **输入给"任务卡"，不给历史。** spawn 的 task 描述要自包含：目标、涉及文件路径、验收标准。不要把主会话最近几轮对话原样粘进去——那是另一种形式的污染。
3. **约定输出格式。** 要求子 Agent 的最终报告只用固定结构（结论 / 产物路径 / 风险），回投主会话的是几十行，而不是几千行。
4. **文件层面需要隔离时拆 agent。** 共享 workspace 时让子 Agent 写独立子目录；任务重的场景干脆定义一个独立 agent（独立 workspace + 精简工具集），spawn 时用 agents 参数指定。
5. **收尾巡检。** 定期 sessions_list 看残留会话，跑完的、僵死的及时清理。session key 统一命名前缀（比如 `sub-<任务名>`），方便识别。注意 `/new` 只重置主会话，子会话残留它管不了。

## 踩坑点

- **任务卡太省。** 只给一句"帮我调研 X"，子 Agent 缺路径和约定，返回一堆不可用的东西，主会话还得再花一轮纠正——净亏损。
- **前台 spawn 超时后重复派生。** 子会话其实还在跑，结果回投了两次，主会话上下文直接翻倍。
- **并发不设上限。** 十个子 Agent 同时打工具 API，限流报错反而全数回灌主会话。
- **子 Agent 继承完整系统提示词和全套工具**，单个子会话的 token 开销比任务本身还大。

## 可复用建议

- 把"窄进宽出"当纪律：输入是自包含任务卡，输出是固定格式结论，主会话只保留决策层信息。
- 子 Agent 并发上限按工具配额反推，宁排队不并发。
- 独立 workspace 或独立子目录，是文件层面最便宜的隔离手段。
- 巡检脚本化：每周跑一次 sessions_list + 清理，成本极低。

## 总结

session 隔离不是什么高级架构，本质是约束信息流：**重过程留在子会话，只让结论回主会话。** OpenClaw 已经把 spawn、session key、独立 workspace 这些机制给足了，剩下的工程习惯——任务卡、输出格式、并发上限、定期巡检——才决定主会话能不能长期保持干净。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-25/15aea328eba5685b.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-25/1189dc9699d61008.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-25/58fa3dd8ec722e70.png)

