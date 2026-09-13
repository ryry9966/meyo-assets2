---
title: OpenClaw 多模型路由实践：GPT 与本地模型的分工切法
feedId: 37431
source: 综合讨论
publishedAt: 2026-09-14
---

## 背景

OpenClaw 的 agent 链路里，模型调用远不止一次问答：规划、工具调用、摘要、抽取、格式化，一个自动化流程动辄几十次调用。全走 GPT 这类云端旗舰模型，成本和延迟都顶不住；全换本地小模型，长链路任务的质量又塌得很快。多模型路由真正要回答的不是“选哪个模型”，而是“哪类调用配哪个模型”。

## 问题

我们内部一个 issue 自动分派流程（读工单 → 分类 → 抽字段 → 起草回复 → 写回）最初全量走云端，月成本和端到端延迟都难看；后来整体切到本地 14B，成本降了，但起草环节的返工率明显上升。结论很直接：链路各环节对模型能力的需求差异很大，必须分级，不能一刀切。

## 做法

**1. 先给任务分三档：**

- 重推理：多步规划、代码生成、复杂工具编排 → 云端旗舰
- 中等：摘要、改写、结构化抽取 → 中档云端或较强的本地模型
- 轻：分类、实体抽取、格式转换、路由判断 → 本地 7B–14B

**2. 建 model profile，技能里只引用 profile，不写死模型名**（示例配置）：

```yaml
profiles:
  planner: { provider: openai, model: gpt-4.1 }
  worker:  { provider: ollama, model: qwen2.5:14b }
routing:
  - match: { task: classify|extract, tokens_lt: 4000 }
    use: worker
  - match: { writes_memory: true }
    use: planner
fallback: [planner]
```

**3. 用结构化校验代替“自信度”：** 本地模型输出先过 JSON schema 校验，失败重试一次，再失败升级到 planner。比让模型自评置信度靠谱得多。

**4. 记 trace 再调参：** 每次调用记录 route、模型、token 数、延迟、重试次数，跑一周后按真实数据调整切分线。

## 踩坑点

- 本地模型的 function calling / JSON 输出不稳，务必开约束解码（grammar / JSON schema），否则工具链会静默断掉，而且很难第一时间发现。
- 上下文窗口：agent 跑几轮后 context 超过本地模型窗口，指令被截断，表现为“突然变笨”。路由前先算 token 预算。
- 延迟要看端到端：本地 GPU 被 embedding 任务占着时，排队时间可能比云端还长。
- 弱模型写的摘要一旦进入长期记忆，会被上游反复消费，质量逐级劣化。我们的硬规则：凡写入记忆的产出，一律走云端。
- 用玩具 prompt 测出的阈值和真实负载分布对不上，切分线只能靠线上 trace 收敛。

## 可复用建议

- 起步用最粗的两分法：只读/分类类调用走本地；写记忆、执行不可逆动作、面向用户的产出走云端。先粗后细。
- 技能里只引用 profile，路由规则集中管理，换模型只改一处。
- 按 stage 设 token 和延迟预算，超限告警，别等账单教育你。
- 每季度重评一次切分线。本地模型迭代很快，半年前的结论大概率过期。

## 总结

多模型路由本质是在质量、成本、延迟、隐私四个约束之间做工程取舍，没有一劳永逸的配置。务实路径是：先两档切分，留好 fallback，把 trace 记全，再用真实数据收敛阈值。OpenClaw 的 profile 加路由钩子足够搭起这套框架，剩下的活是持续看数据、持续调。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-14/0d4d33af692c968f.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-14/f8babaaa5e285449.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-14/9e4a5b34d99015be.png)

