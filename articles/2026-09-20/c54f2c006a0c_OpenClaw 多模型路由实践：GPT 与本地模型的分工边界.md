---
title: OpenClaw 多模型路由实践：GPT 与本地模型的分工边界
feedId: 38274
source: 综合讨论
publishedAt: 2026-09-20
---

## 背景

多数 OpenClaw 部署的起点都一样：一个主 agent，一堆 MCP 工具和插件，配置里只填了一个模型。能跑，但账单和延迟会慢慢告诉你这不划算；而涉及私有数据的场景，全走云端还多一层合规压力。

## 问题

两个常见的失败模式：

- **全云**：摘要、意图分类、格式抽取这类"低难度高频率"任务也在烧 GPT，月成本失控，高峰期延迟叠在整条 agent 链路上。
- **全本地**：多步规划和工具编排明显变弱，MCP 参数填错、任务半路跑偏，而且失败得很安静，不容易第一时间发现。

结论不是二选一，而是按任务分工。

## 做法

**1. 先盘点任务形态。** 把 agent 实际干的事列出来：规划 + 工具编排（需要强模型）、摘要 / 分类 / 结构化抽取（本地够用）、代码生成（强模型）、涉敏数据处理（本地是硬约束）。

**2. 定义双档 model profile。** 在 OpenClaw 配置里把模型分成两档，例如：

```yaml
models:
  strong: gpt-4.1
  local: qwen2.5:14b        # ollama / vllm
routing:
  planner: strong
  summarize: local
  extract: local
fallback: [local, strong]
```

**3. 路由粒度放在 subagent 层，不做全局开关。** 主 agent 和 planner 留在强模型上，把线程摘要、意图分类这类内部子任务下沉到本地，收益最大、风险最小。

**4. 写一条升级链。** 本地先跑，校验失败（JSON 解析失败、工具参数不合法）就升级到 GPT 重试，同时把失败样本落日志，供后续评估。

**5. 建一个 20～50 条的回归集。** 每次调整路由跑一遍，只看三个数：工具调用成功率、P95 延迟、单日成本。

## 踩坑点

- 本地模型的 function calling 兼容性参差，务必开 structured output 或语法约束，否则 MCP 参数错得悄无声息。
- 上下文长度：MCP 工具 schema 加对话历史很快吃满 8k/32k，本地小模型容易截断、丢工具定义。
- 流式输出中途 fallback 会产生重复消息，重试前先判断是否已有内容发出。
- Q4 量化做摘要还行，做多步规划掉点明显，别拿 benchmark 分数线性外推。
- 中文任务里 Qwen 系本地表现明显好于 Llama 系，别只看英文榜单选型。
- 本地模型冷启动首 token 可能要十几秒，网关超时记得放宽。

## 可复用建议

- 按"任务形态"路由，不要按模型名字路由。
- 只做两档（local / strong），五层路由的维护成本不划算。
- 每条路由打 tag，记录成本、延迟、成功率，一周数据就够你判断路由对不对。
- fallback 逻辑收进 OpenClaw 配置，别散落在各插件的代码里。
- 敏感字段留在本地是合规底线，不是省钱手段。

## 总结

多模型路由的目标不是省每一个 token，而是让任务形态和模型能力匹配。从两档加一条升级链开始，用回归集的数据决定哪些任务可以下沉。多数场景下，摘要、分类、抽取下沉能砍掉六成以上的云调用，而规划留在强模型上，稳定性几乎不受影响。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-20/da2277b9b903470f.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-20/f99c66fc99876ac9.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-20/00dfd3d3b99bdd77.png)

