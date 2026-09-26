---
title: OpenClaw 多模型路由实践：哪些活给 GPT，哪些活留给本地模型
feedId: 39117
source: 综合讨论
publishedAt: 2026-09-26
---

## 背景

OpenClaw 的 agent 流水线里，模型调用的性质差异很大：任务规划、多步推理这类调用一天可能只有几十次，但对模型能力要求高；而工具输出的摘要压缩、结构化抽取、意图分类这类调用一天成千上万次，能力要求不高但很烧 token。全部走 GPT，成本和延迟都难受；全部走本地模型，复杂任务的成功率又明显下滑。多模型路由的本质，就是把这两类负载拆开。

## 问题

我们最初的配置是“一刀切”：所有请求默认走云端 GPT，想省钱就把 agent 整体切到本地 14B 模型。结果两个极端都不行——前者月账单里 70% 花在“把工具返回的 JSON 摘要一下”这种事上；后者 agent 多步任务成功率从 82% 掉到 61%，而且坏得很安静，不跑评测根本发现不了。问题不在模型，在于没有路由策略。

## 做法

1. **给任务打标**。在插件层把模型调用分成三类：`plan`（规划/推理）、`transform`（摘要、抽取、格式化）、`classify`（分类、rerank、路由判断）。
2. **按三维写路由规则**：任务类型 + 输入长度 + 是否含敏感数据。核心思路是 plan 走 GPT，transform 和 classify 走本地模型（Ollama/vLLM 挂 7B~14B），输入超过本地上下文窗口的强制回云端。
3. **配置 fallback 链**：本地超时（我们设 3s）或 schema 校验失败时升级到 GPT，并记录路由原因。
4. **评测守门**：每类任务维护 30~50 条 golden case 接入 CI，路由改动必须过评测再上线。

配置大致长这样：

```yaml
routing:
  rules:
    - match: { task: plan }
      model: gpt-4o
    - match: { task: transform, tokens: "<=6000" }
      model: local/qwen2.5-14b
    - match: { task: classify }
      model: local/qwen2.5-7b
  fallback: gpt-4o-mini
  local_timeout: 3s
```

5. 上线后盯三个指标：每条路由的命中率、fallback 率、单任务平均成本。

## 踩坑点

- **工具调用格式不兼容**：本地模型输出的 function call JSON 经常缺字段或不合 schema，必须在插件层加 schema 校验加一次修复重试，否则 fallback 会被频繁触发，反而更贵。
- **上下文窗口陷阱**：按字符数估 token 会低估，中文建议按 1.6~1.8 倍估；超限请求直接路由走，别让本地模型硬吃然后截断。
- **延迟错觉**：4060 这类卡跑 14B 量化模型，单请求延迟可能比 GPT API 还高。本地模型的优势是成本和隐私，不一定是速度。
- **embedding 不要混用**：流水线里向量维度要一致，别一半本地一半云端，召回率会莫名其妙地掉，还很难归因。

## 可复用建议

- 路由规则保持“少而硬”，先只分 plan / 其他两档跑稳，再逐步细分。
- fallback 不是免费的，设日预算上限，防止本地模型退化时账单失控。
- 每次路由决策都带 reason 字段写日志，事后归因全靠它。

## 总结

多模型路由不是“能用便宜的就用便宜的”，而是把 agent 的调用按能力要求和数据敏感度分层。我们的结果：token 成本降了约 55%，p95 延迟基本持平，任务成功率回到 81%。先打标、再路由、评测守门、fallback 兜底——这四步的顺序不要乱。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-26/b53d8f7e28b1deab.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-26/1986c175c35bfbcd.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-26/f6f10340aba890e2.png)

