---
title: 在家跑大模型：给 Agent 用户的本地 LLM 部署实录
feedId: 38238
source: 综合讨论
publishedAt: 2026-09-20
---

## 背景

作为 OpenClaw 用户，我们的 agent 常年在线：处理消息、跑自动化、调 MCP 工具。所有 token 都走云端 API，有两件事让我不太踏实：家庭对话和自动化上下文全存在别人服务器上；高频轻任务（意图分类、摘要、字段抽取）的成本月积月累。把一部分推理搬回家，是很自然的下一步。

## 问题

先说结论：**“跑起来”和“能用”是两回事**。家用硬件下，核心矛盾有三个：

1. 显存/内存预算：模型权重只是大头之一，KV cache 随上下文线性增长；
2. 吞吐：agent 一轮任务动辄五六次模型往返，8 tok/s 和 60 tok/s 是“难受”和“流畅”的区别；
3. 工具调用可靠性：本地小模型的 function calling 远不如旗舰云模型稳，而这恰是 agent 依赖的核心。

## 做法

我的环境：i5-12700 + RTX 4070 Ti 12GB 台式机，外加一台 16GB 统一内存的 Mac mini。

**1. 盘点硬件，倒推档位。** Q4_K_M 量化下，7–8B 模型约占 5GB，14B 约 9GB，32B 约 20GB，之上还要给 KV cache 预留 20–30%。12GB 显存的现实上限就是 14B @ 16k 上下文。

**2. 选引擎。** 省事用 Ollama，NVIDIA 卡要并发上 vLLM，Mac 用 Ollama 或 LM Studio。共同点是都暴露 OpenAI 兼容端点。

**3. 选模型。** 别看榜单，看自己 agent 的真实任务。我固定为 Qwen3-14B Q4 做本地主力、8B 干高频杂活，理由只有一个：中文指令 + 工具调用场景下格式遵循够稳。

**4. 接入 OpenClaw。** 模型配置里加一个自定义 provider，baseUrl 指向 `http://127.0.0.1:11434/v1`，apiKey 填占位符即可。再用路由/fallback 分层：意图识别、会话摘要、JSON 抽取走本地；复杂推理和长上下文走云端。

**5. 压测。** 从历史会话取 50 条真实样本，记录 TTFT、tokens/s、工具调用成功率三项，再决定哪些任务正式迁移。

## 踩坑点

- **Ollama 默认上下文只有 4k。** 系统提示加工具描述轻松超限，超限不报错、静默截断，表现为“模型突然变笨”。务必显式设 `num_ctx`。
- **量化不是免费的。** Q4 下小模型输出 JSON 偶发字段缺失、格式漂移。给结构化输出加 schema 约束（Ollama structured outputs / llama.cpp GBNF），失败率明显下降。
- **别把全部 MCP 工具喂给本地模型。** 十几个 server 的工具描述能吃掉几千 token，小模型面对长工具列表的选择准确率会跳水。只挂高频的三五个。
- **并发会排队。** Ollama 默认串行处理请求，多会话同时来会互相堵，要么调并行参数，要么接受延迟。
- **“能跑”不等于“能用”。** 16GB Mac 跑 32B 没问题，但 10 tok/s 的 agent 循环会让你怀疑人生。

## 可复用建议

1. **分层路由是最大收益点**：别追求全本地，让小模型干它擅长的杂活；
2. **用真实日志建 eval 集**：换模型只看工具调用成功率和 tokens/s，不看榜单；
3. **配置即代码**：docker-compose + 固定模型 tag，换机十分钟恢复；
4. **盯三个指标**：TTFT、tokens/s、工具调用失败率。出问题先看数据，再猜原因。

## 总结

本地 LLM 已经足够撑起 agent 工作流里“高频、轻量、敏感”的那一层，但替代不了云模型的硬推理能力。务实的答案是混合部署：12GB 显存 + 14B Q4 做本地层，云端兜底，用真实任务数据划清边界。这套东西一个周末能搭完，建议动手试一次。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-20/dcaefb07510ae3b5.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-20/f4df3f9ef0a2884d.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-20/83246ea66f12eadf.png)

