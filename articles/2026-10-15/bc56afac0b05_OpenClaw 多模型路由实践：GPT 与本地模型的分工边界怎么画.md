---
title: OpenClaw 多模型路由实践：GPT 与本地模型的分工边界怎么画
feedId: 41584
source: 综合讨论
publishedAt: 2026-10-15
---

## 背景

我们组用 OpenClaw 搭了一个运维 Agent，技能大概三类：多步规划 + MCP 工具编排、批量日志摘要、结构化信息抽取。最初为了省事，所有请求都走同一个旗舰云端模型。跑了一个月，两个问题压不住了：一是账单，八成以上的请求其实是“摘要、抽取、格式化”这类低复杂度任务，用旗舰模型属于高射炮打蚊子；二是部分日志字段涉及内网信息，合规上不允许出本地环境。

## 问题

把云端模型整体换成本地模型试过一周，结果更糟：多步工具调用时 function call 的 JSON 经常残缺，Agent 中途断链，重试率翻倍。所以真正的题目不是“选哪个模型”，而是给出一条清晰的分工边界——什么请求路由到 GPT，什么请求留在本地。

## 做法

分四步：

1. **统一接入层**。本地模型用 vLLM（Ollama 也行）暴露 OpenAI 兼容接口，和云端模型一起注册进 OpenClaw 的模型池。路由器只面对统一 schema，不感知底层差异。
2. **给技能打标**。在 skill 元数据里加 route hint：`plan` / `toolchain` 标云端，`summarize` / `extract` / `transform` 标本地。路由质量取决于这一步的标签是否诚实。
3. **两维规则路由**。不要上来就训练分类器。先用两个维度：任务标签 + 预估 token。超过 8k 上下文、或多跳工具链的请求，即使标签是“简单”也强制走云端。
4. **降级链 + 观测**。云端超时或 429 时降级到本地，路由决策（命中哪条规则、最终用了哪个模型）写入 trace，每周拉一次路由命中率、降级次数和 p95 延迟。

核心配置收敛下来十几行：

```yaml
router:
  rules:
    - when: { tag: [plan, toolchain], route: cloud-gpt }
    - when: { tag: [summarize, extract], route: local-qwen }
    - when: { est_tokens_gt: 8000, route: cloud-gpt, override: true }
  fallback: [cloud-gpt, local-qwen]
```

## 踩坑点

- **本地模型 function call 不稳**。开了 schema 约束 + 服务端校验重试后，断链率从 12% 降到 2%。没加约束就上生产，是最大的一次翻车。
- **显存假象**。本地模型标称 32k 上下文，实际长输入时 KV cache 直接爆显存。按实测把 `max_context` 砍到能稳定跑的值，别信纸面参数。
- **别用 LLM 当路由器**。试过让小模型做请求分类，等于每步多一次调用，延迟几乎翻倍，而且分类本身也会错。规则先行，分类器是后话。
- **提示词不兼容**。为 GPT 调的长 system prompt 直接喂本地模型，效果明显劣化。给本地模型单独维护一份精简版。
- **静默降级**。降级不告警，两周后才发现三成“云端任务”悄悄跑在本地，质量下滑没人知道。降级必须打点。

## 可复用建议

- 起步只看两个维度：任务类型 + token 量，跑稳后再考虑更多特征。
- 固定映射比动态路由靠谱：高频简单任务锁死本地，规划与工具编排锁死云端，灰区才走规则。
- trace 里显式记录 `route` 字段，这是复盘和成本归因的唯一依据。
- 每次升级本地模型版本，重跑一遍工具调用回归集，不要裸升。

## 总结

多模型路由的价值不在“省了多少钱”，而在把昂贵的智力精准留给少数难步骤。OpenClaw 的模型池 + 规则路由足以覆盖大部分场景，关键是技能打标要诚实、降级要可见、本地模型的工程约束（schema 校验、上下文实测）不能省。先把分工边界画清楚，再谈智能化。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-15/8615576e79034785.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-15/de40d6869826264a.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-15/d61d240f513f7e4c.png)

