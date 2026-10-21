---
title: OpenClaw 多模型路由实战：什么时候用 GPT，什么时候用本地模型
feedId: 42233
source: 综合讨论
publishedAt: 2026-10-22
---

## 背景

Agent 跑起来之后，大多数团队遇到的第一个现实问题是成本：所有任务全打到 GPT 上，账单肉眼可见地涨。反过来全切本地模型（vLLM/Ollama 跑个 14B），插件调用失败率、复杂任务返工率又明显上升。所以多模型路由的难点从来不是"要不要分流"，而是**按什么维度分流**。

## 我们的问题

我的场景是一个内部数据处理 Agent，日均几百次任务，大致三类：

1. 意图识别、字段抽取这类轻任务——占 60% 调用量，全走云模型时钱花得最冤；
2. 涉及公司内部数据的总结分析——合规要求根本不能出网；
3. 多步工具编排和代码生成——本地模型质量扛不住。

## 做法

用 OpenClaw 的 router 按规则匹配 model pool，逻辑最终收敛成三层：

```yaml
routes:
  - match: { task: classification | extraction }
    model: local-qwen-14b
  - match: { data: internal }
    model: local-qwen-14b
    policy: no_fallback_cloud
  - match: { task: orchestration | coding }
    model: gpt-4o
    fallback: [local-qwen-72b]
```

三个关键点：

1. **在任务源头打标，不要用 LLM 判路由。** Agent 发起子任务时就在 metadata 里带 `task` / `data` 标签，router 只做纯规则匹配。用小模型判断"这个任务难不难"本身也要花钱，还引入新的错误源。
2. **按能力分层，不按模型名写死。** 配置里只写 `frontier / mid / local` 三档，具体模型放 pool 里，换模型不动路由规则。
3. **敏感数据路由加 `no_fallback_cloud`。** 本地失败走重试和任务降级（缩小粒度），绝不回退到云。

## 踩坑点

- **工具调用格式不兼容。** 本地小模型对 function calling 支持参差不齐，同一个 MCP 工具在 GPT 上参数填得好好的，到 14B 上就开始乱来。强依赖工具调用的任务别路由到本地。
- **上下文长度错配。** 云 128k、本地 32k，router 层必须做截断或分段，否则任务悄悄劣化，表象是"模型变笨了"，很难排查。
- **本地延迟的错觉。** 跑分上 14B 很快，但冷启动、KV cache 未命中、和别的进程抢显卡时，P95 反而比云模型高。路由决策看 P95，别看理论吞吐。
- **回退循环。** 本地失败→回退云→云超时→又重试本地，延迟和账单双爆。回退链必须加熔断：同一任务 5 分钟内最多回退一次。

## 可复用建议

1. 调规则前先攒 50~100 条真实任务做小评测集，否则你不知道分流后质量掉了多少。
2. 每次路由决策记录 `route_to + reason`，跑一周拉出来复盘，该合并的合并、该删的删。
3. 给单任务设成本上限，超限走低价档加降级，而不是直接报错。
4. 路由规则保持朴素。规则能解决的事不要上"智能路由"——排障时，可解释性比聪明值钱得多。

## 总结

多模型路由本质是成本、隐私、质量的三角取舍。我的经验值：轻任务和敏感数据坚决走本地，编排和代码走云模型，回退链一定带熔断。路由层保持简单、决策留痕，比任何动态路由算法都更扛得住生产环境。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-22/e6d35255a1094e9a.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-22/850a903559d9d8f7.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-22/b0e6af415b577afa.png)

