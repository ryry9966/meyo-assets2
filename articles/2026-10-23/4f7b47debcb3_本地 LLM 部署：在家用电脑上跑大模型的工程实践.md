---
title: 本地 LLM 部署：在家用电脑上跑大模型的工程实践
feedId: 42324
source: 综合讨论
publishedAt: 2026-10-23
---

## 背景

接入 OpenClaw 这类 agent 框架后，我对云端 API 的两个不满越来越明显：自动化任务 7×24 调用，账单随工具循环次数线性上涨；本地文件、家庭数据走云端处理，隐私上始终不踏实。于是花了几个周末，把一部分负载迁到了本地推理。

## 问题

本地跑 chat 很容易，本地跑 agent 难。区别在于 agent 需要：稳定的 tool calling、足够长的上下文（系统提示 + MCP 工具 schema 轻松上万 token）、可控的结构化输出。这三件事在消费级硬件上互相挤压，是整个实践的核心矛盾。

## 做法与步骤

我的环境：Ryzen 7 + 32GB 内存 + RTX 4060 Ti 16GB，供参考。

1. **先盘硬件再选模型。** Q4_K_M 量化下的经验值：7-8B 约占 5GB，14B 约 9GB，32B 约 20GB。显存不够可以 CPU offload，但速度明显打折。16GB 显存跑 14B 全量 offload 比较舒服；纯 CPU 建议止步 7-8B。
2. **选对模型比选大模型重要。** 中文 + 工具调用场景，优先挑官方声明支持 function calling 的型号（Qwen 系列实测最省心），下载后先单独测工具调用，再接入框架。
3. **运行时用 Ollama 或 llama.cpp server**，都是 OpenAI 兼容接口。OpenClaw 侧只需改 provider 的 base_url 和模型名，插件和 MCP 配置原样不动。LM Studio 适合桌面端快速调试。
4. **显式设置上下文窗口。** Ollama 默认 num_ctx 很小，必须手动调到 8192 以上，并按需开启 flash attention 和 KV cache 量化。
5. **接入后做三个冒烟测试**：单次工具调用、多轮工具链、长文档问答。都跑通再谈日常使用。

## 踩坑点

- **上下文静默截断**：系统提示 + 工具 schema 超过 num_ctx 时不报错，模型只是"变笨"——忘指令、编参数。翻日志看 usage 才能定位。
- **工具调用模板不匹配**：llama.cpp 走 chat template，部分模型没有原生 tool 标记，function call 会以纯文本吐出，框架解析失败。选型时确认 runtime 支持该模型的模板。
- **KV cache 吃显存**：上下文调大后 16GB 卡直接 OOM。开 flash attention + KV cache 量化通常能救回来。
- **thinking 模型拖慢自动化链路**：每次工具循环都先输出一段推理，10 次调用的任务延迟翻几倍。跑自动化时关掉推理输出或换非 thinking 版本。
- **温度影响结构化输出**，工具调用任务建议压低温度并强制 JSON schema。

## 可复用建议

- **分层路由**：高频简单任务（意图识别、摘要、格式化）走本地小模型，复杂推理走云端。OpenClaw 配两个 provider 按任务路由，账单能降一个量级，本地模型也不用硬扛最难的问题。
- **用自己的真实工作流做 benchmark**，别信榜单。重点记录三个数：tokens/s、端到端任务耗时、工具调用成功率。
- **模型与参数配置进 git 固定版本**，出问题可回滚、可复现。

## 总结

本地部署的门槛不在"跑起来"，而在"跑稳 agent"。核心是三件事：选对支持工具调用的模型、显式管好上下文、用真实任务验证。先让本地小模型接管高频廉价任务，再逐步扩大它负责的边界，是家庭硬件上最稳的路径。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-23/43a4b3ac4269c845.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-23/7096260f8fad7c5f.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-23/b84267cb6d009822.png)

