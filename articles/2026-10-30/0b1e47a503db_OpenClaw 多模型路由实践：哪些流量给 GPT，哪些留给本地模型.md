---
title: OpenClaw 多模型路由实践：哪些流量给 GPT，哪些留给本地模型
feedId: 43041
source: 综合讨论
publishedAt: 2026-10-30
---

## 背景

Agent 跑一段时间后，流量分布会很不对称：大量调用其实是摘要、格式转换、简单信息抽取，真正需要复杂推理和多工具编排的可能不到两成。这些请求全走 GPT 级云端模型，成本和延迟都在浪费；反过来全压到本地 7B/14B，复杂任务成功率又明显掉。OpenClaw 的路由策略（routing policy）解决的就是这个错配。

## 问题

难点不在“能不能路由”，而在三个工程问题：

1. 路由粒度怎么定——按请求、按工具，还是按任务阶段？
2. 本地模型的 structured output 和 function calling 不稳，失败后怎么兜底？
3. 路由决策本身怎么验证是对的，而不是拍脑袋？

## 做法

**第一步，给调用分类。** 按任务性质分，不按模块分：规划/多步推理类、工具调用类、格式化/摘要类、embedding 类。先跑一周全云端，把 trace 记下来统计分布，这一步别跳过，后面所有规则都靠它。

**第二步，规则优先、级联兜底。** 用确定性规则处理大部分流量（命中哪些工具、上下文长度阈值、是否涉及私有数据），规则覆盖不了的走 cascade：本地先跑，schema 校验失败或超时，重试一次后升级云端。配置大致是：

```yaml
routing:
  rules:
    - match: { task: format, ctx_tokens_lt: 8000 }
      model: local/qwen2.5-14b
    - match: { private_data: true }
      model: local/qwen2.5-14b
  cascade:
    primary: local/qwen2.5-14b
    fallback: gpt-4o
    escalate_on: [schema_fail, timeout]
```

**第三步，加护栏。** 本地输出一律过 JSON schema 校验；带 tool_calls 的请求要求模型支持原生 function calling，不支持就直接走云端，别硬转文本协议。路由日志必须带 task_id 和命中的规则，否则没法复盘。

## 踩坑点

- 本地小模型的 function calling 参数经常格式出错，schema 校验不是可选项，是必选项。
- 别拿 logprob 当“置信度”做升级判断，噪声太大。先用任务级规则，攒够真实数据再考虑精细化。
- GPU 被 embedding 或其他任务占满时，本地 fallback 反而比云端慢。级联里必须设超时，超时即升级，不要原地重试。
- 上下文超过本地窗口会被静默截断，任务看起来“成功”了其实输入缺了一块，规则里要加长度判断。
- fallback 只升一级。本地→小云→大云的连环重试会把 P95 延迟打爆，宁可失败得快一点。

## 可复用建议

- 路由策略做成配置而非硬编码，支持灰度：新策略先吃 10% 流量，看两天再放量。
- 用真实 trace 回放做评测，对比“全云端 vs 路由后”的成功率和 P95 延迟，用数据决定哪些流量下沉，而不是凭感觉。
- 隐私敏感场景（内部文档、代码库）直接规则强制本地或私有化部署，不要依赖 cascade 的运气。
- embedding 模型一旦选定就不要在路由里切换，索引和查询必须同源，换模型的迁移成本远大于省的那点推理费。

## 总结

多模型路由不是“哪个模型强用哪个”，而是把任务分层：确定性规则吃掉大部分简单流量，cascade 只兜底不确定的部分，云端留给真正需要推理深度和工具编排的请求。先把日志和评测跑起来，再调策略——路由的收益要能算得清，才算真的在做工程。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-30/27d538e77f40e7ea.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-30/92b5e6e3a71816d5.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-30/5e20f4cba9d833d5.png)

