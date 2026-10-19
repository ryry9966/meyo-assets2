---
title: 本地 LLM 部署：在家用电脑上跑大模型的实践指南
feedId: 42035
source: 综合讨论
publishedAt: 2026-10-19
---

## 背景

云端 API 用得越多，两个问题越明显：一是成本，Agent 循环动辄几十次调用，token 消耗很可观；二是隐私，日志、代码、内部文档这类数据不希望出本机。对 OpenClaw 这类自动化场景，本地模型的价值不在于"替代云端大模型"，而在于承接高频、低难度的调用，把云端额度留给真正需要推理深度的步骤。

## 问题

但本地部署的决策点比想象中多：显存够不够、选哪个参数量级、量化到什么程度、推理框架怎么选、Agent 依赖的 function calling 是否可靠。多数教程只讲"能跑起来"，不讲"能不能撑住一个 Agent 循环"。

## 做法

**1. 摸清硬件底牌。** 显存是硬约束，估算公式：模型占用 ≈ 参数量 × 量化位宽 ÷ 8，再留 20%-30% 给 KV cache。参考值：7B Q4 约 5GB，14B Q4 约 9GB，32B Q4 约 20GB。纯 CPU 也能跑 7B，但 10 tok/s 以下的吞吐对 Agent 循环基本不可用。

**2. 选模型和量化。** 优先选官方支持工具调用的指令模型，Qwen2.5 / Qwen3 系列在中文场景和 function calling 上表现稳定。量化建议 Q4_K_M 起步——为省显存压到 Q2，工具调用的 JSON 输出会明显劣化，Agent 侧解析直接失败。

**3. 部署推理服务。** Ollama 或 llama.cpp 起步足够，两者都暴露 OpenAI 兼容接口。关键配置三处：`n_gpu_layers` 尽量拉满（留 1-2GB 余量）；ctx 长度按实际需要设，32k 的 KV cache 能多吃掉几个 GB；开启 flash attention。跑通后用 curl 打一下 `/v1/chat/completions` 确认接口正常。

**4. 接入 Agent/MCP。** 把 base_url 指向 localhost 即可，框架侧几乎零改动。先用一个两步小任务验证闭环：调一次工具、总结结果，确认模型能正确产出工具调用格式、输出里没有干扰解析的多余文本。

## 踩坑点

- **ctx 设太大导致 OOM**：KV cache 随上下文线性增长，8k 够用就别开 128k。
- **WSL2 下 GPU 利用率异常**：先确认驱动和 CUDA 版本匹配，必要时换原生 Windows 版 Ollama 排除变量。
- **思考型模型输出 think 标签**：需要剥离后再喂给解析器，否则工具调用解析随机失败。
- **温度和 stop 参数默认值与云端不同**：工具调用建议 temperature ≤ 0.3。
- **模型热切换加载慢**：GGUF 支持 mmap，但冷启动读盘仍需几十秒，Agent 场景尽量固定模型。

## 可复用建议

- 把"本地/云端"做成配置项而非代码分支，同一套 Agent 逻辑两套 endpoint 随时切换。
- 用自己的真实 Agent 任务做压测，别参考通用榜单——你要的是工具调用成功率，不是答题分数。
- 做分层路由：简单调用走本地 7B，复杂推理走云端大模型，用 token 日志统计各步骤用量来定切分点。
- 推理服务用 Docker 固化，配置写进 compose 文件，换机可复现。

## 总结

消费级硬件跑 7B-32B 模型在今天已是成熟方案，但它的定位是"高频常规调用的承接者"，不是全能替代。核心动作就三个：按显存选模型、按任务定量化、按 Agent 吞吐验收。先跑通一个最小闭环，再逐步扩大本地承接比例，比一上来追求全本地化务实得多。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-19/d5929535c66a4dce.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-19/3b07d8314e85e5e8.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-19/416852d455f2fd15.png)

