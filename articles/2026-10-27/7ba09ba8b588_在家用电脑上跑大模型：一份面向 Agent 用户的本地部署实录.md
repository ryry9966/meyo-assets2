---
title: 在家用电脑上跑大模型：一份面向 Agent 用户的本地部署实录
feedId: 42698
source: 综合讨论
publishedAt: 2026-10-27
---

# 背景

做 Agent 自动化这几年，我的工作流里高频调用 LLM：整理剪贴板、批量分类文件、跑定时任务。全走云端 API，一个月账单不小，而且剪贴板、邮件这类内容并不想出本机。于是花了两个周末，把手头的主力机接了一套本地推理，过程可复现，记录如下。

# 问题

家用电脑跑大模型，核心矛盾有三个：

1. 显存/内存有限，模型选大了直接 OOM；
2. Agent 场景吃 function calling，很多小模型工具调用不稳定；
3. 长上下文是内存黑洞，KV cache 常被低估。

# 做法与步骤

**1. 盘点硬件，倒推模型规模。** 粗略估算：权重显存 ≈ 参数量 × 每参数字节数。Q4_K_M 量化约 0.6 byte/参数，7B 约 4.5GB，14B 约 9GB，再留 2–4GB 给 KV cache 和运行时。我的 12GB 显存卡最终选了 14B 的 Q4；纯 CPU 机器跑 7B Q4 也能到每秒十几个 token，够用。

**2. 选推理框架。** 图省事用 Ollama，一条命令拉模型；要细调参数用 llama.cpp；追求吞吐再上 vLLM。家用场景 Ollama 足够。

**3. 选对模型家族。** Agent 用户优先挑官方支持工具调用的系列（如 Qwen、Llama 3.1+），否则 MCP 工具调用基本残废。别只看排行榜的聊天分数。

**4. 接进 OpenClaw。** Ollama 暴露 OpenAI 兼容接口，把 base_url 指到 `localhost:11434`，模型名写死并带 tag（如 `qwen2.5:14b-q4`），agent 其余配置不动。

**5. 显式设置上下文长度。** 这步最容易被忽略，见下文踩坑点。

# 踩坑点

- **上下文默认值太小。** 默认 `num_ctx` 很小，Agent 一挂 MCP 工具描述加对话历史，System prompt 直接被截断，表现为"模型忘了自己有哪些工具"。显式设到 8k–16k。
- **KV cache 吃显存超预期。** 上下文翻倍，KV cache 近似翻倍。OOM 时先降 ctx，别急着换小模型。
- **量化太狠伤工具调用。** Q2/Q3 聊天看起来还行，但 function calling 的 JSON 合法率明显下滑，至少 Q4_K_M 起步。
- **GPU 卸载层数别一次拉满。** `n_gpu_layers` 从 2/3 开始试，OOM 就回退几层，混合卸载往往比纯 CPU 快得多。
- **小模型 JSON 会出错。** 在插件层加 schema 校验 + 一次重试，失败率能压到可接受范围。

# 可复用建议

- 建一个 10–20 条真实任务的小评测集（含 3–5 条工具调用），换模型前跑一遍，比看榜单靠谱。
- 用 Modelfile 把 temperature、num_ctx 固化，避免不同终端配置漂移。
- 做混合路由：本地跑高频简单任务（分类、摘要、格式转换），复杂推理和长文降级到云端 API，在 OpenClaw 里按任务类型分流即可。
- 留日志记录 token/s 和调用失败率。本地模型的"退化"是渐进的，没有数据很难发现。

# 总结

本地 LLM 已经从玩具变成可用的生产组件，前提是三件事：按显存倒推模型规模、认准支持工具调用的模型家族、显式管好上下文与量化等级。它不会替代云端大模型，但作为 Agent 的"日常兜底"，省下的成本和隐私收益非常实在。具体的显卡和模型搭配问题，欢迎评论区交流。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-27/5e20ea8aa554416f.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-27/168e8251d918b1f8.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-27/2ba4a38fe44ccdb9.png)

