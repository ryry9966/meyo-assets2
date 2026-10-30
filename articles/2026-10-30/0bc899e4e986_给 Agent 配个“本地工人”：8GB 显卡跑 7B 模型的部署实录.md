---
title: 给 Agent 配个“本地工人”：8GB 显卡跑 7B 模型的部署实录
feedId: 43044
source: 综合讨论
publishedAt: 2026-10-30
---

## 背景

跑 Agent 的人大多绕不开两件事：token 账单随循环次数线性上涨；个人笔记、相册、密钥这类数据不想出境。把高频低难度的请求挪到本地，正好是这两个问题的交集。我的场景很典型：OpenClaw 里挂着十几个 MCP 工具，其中摘要、分类、向量检索占了请求量的七成以上——这类活儿交给本地模型正合适，规划和复杂工具链继续走云端。本文记录在一台 8GB 显存台式机上部署 7B/14B 模型并接入 Agent 流程的完整过程。

## 问题

动手前明确了三个要验证的问题：

1. 8GB 显存到底能塞下什么级别的模型，上下文还能留多少；
2. 小模型做工具调用的 JSON 输出是否够稳；
3. Agent 一轮任务动辄十几次请求，本地吞吐会不会拖后腿。

## 做法与步骤

**第一步：算显存账。** Q4_K_M 量化约 0.6 GB/参数，7B 模型本体约 4.2GB，剩给 KV cache 的不足 4GB。结论：7B 可跑但上下文要克制，14B 只能部分 offload 到内存。先 `nvidia-smi` 确认系统本身没偷吃显存。

**第二步：装推理服务。** 选 Ollama，图的是 OpenAI 兼容端点开箱即用；追求吞吐可换 llama.cpp 的 `llama-server` 或 vLLM。顺手打开 `OLLAMA_FLASH_ATTENTION=1` 和 `OLLAMA_KV_CACHE_TYPE=q8_0`，KV cache 体积直接减半。

**第三步：拉模型，显式设上下文。** `ollama pull qwen3:8b`，进入配置把 `num_ctx` 设为 16384；同时拉 `bge-m3` 做本地 embedding。

**第四步：接入 Agent。** 把框架的 `base_url` 指向 `http://localhost:11434/v1`，key 随意填。MCP 工具对模型无感知——工具 schema 最终都进 prompt，本地模型一样能调，只需验证它读得懂 schema。

**第五步：跑冒烟测试。** 写了个 50 次调用的脚本，固定一组 MCP 工具（读文件、抓网页），统计 JSON 解析成功率和 tokens/s。

## 踩坑点

- **默认上下文截断是最阴的坑。** Ollama 默认 `num_ctx` 很小（2k/4k），工具定义一多就被静默截断，模型表现得像“失忆”。必须显式设置；且注意 32k 上下文的 KV cache 还能再吃 2–3GB。
- **并发请求会放大显存。** Agent 框架偶尔并发，`OLLAMA_NUM_PARALLEL` 大于 1 时显存按倍数涨，8GB 卡建议保持 1。
- **小模型的 JSON 会漂。** 7B 工具调用成功率初期仅 78%，把 temperature 降到 0.2、请求里强制 `format=json` 后稳定到 95% 以上；更严格的场景可上 llama.cpp 的 json_schema 约束。
- **别拿统一内存和独显比速度。** Mac 内存大不等于带宽够，M 系列跑 14B 能跑，但 tokens/s 会明显低于同量化的桌面卡。

## 可复用建议

1. 先跑 7B Q4 基线，让测试数据决定要不要上更大模型，别反过来。
2. 维护一张模型登记表：模型名、量化、num_ctx、显存占用、tokens/s、工具调用通过率，换模型只看这张表。
3. 端点做成配置项而非硬编码，云/本地切换应该是一行改动。混合路由：本地接摘要/分类/embedding，云端接规划和多步工具链。
4. 换 embedding 模型记得重建向量索引，维度不一致导致的检索劣化非常隐蔽。

## 总结

本地 LLM 不是云端的替代品，而是给 Agent 补一块高频、廉价、数据不出门的容量层。把它当基础设施对待：有冒烟测试、有指标表、有回退路径。8GB 显存跑 7B Q4 + 16k 上下文这套组合，在我这边稳定承接了日常流程七成以上的请求，性价比足够。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-30/b7c71dd318258651.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-30/c4da196515b88afd.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-30/56128abae1eb9722.png)

