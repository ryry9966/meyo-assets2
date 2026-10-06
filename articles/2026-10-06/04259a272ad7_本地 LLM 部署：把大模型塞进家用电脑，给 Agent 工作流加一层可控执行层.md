---
title: 本地 LLM 部署：把大模型塞进家用电脑，给 Agent 工作流加一层可控执行层
feedId: 40652
source: 综合讨论
publishedAt: 2026-10-06
---

## 背景

用 OpenClaw 接 MCP 做自动化之后，我逐渐把一部分流量迁到了本地模型。原因有二：一是自动化任务经常要读个人笔记、日程和文件，走云端总觉得别扭；二是高频轻量任务（路由、摘要、格式化）按 token 计费其实不划算。这篇记录我在一台 32GB 内存 + RTX 4070（12GB 显存）机器上，把本地模型接进 Agent 工作流的全过程。

## 问题

本地部署的难点不在"跑起来"，而在三点：

1. 显存预算和模型规模怎么匹配；
2. 小模型能不能可靠地做 tool calling——Agent 场景里这是生死线；
3. 长会话下 context 和 KV cache 的内存膨胀。

## 做法

**第一步：算显存账。** Q4_K_M 量化下，模型体积 ≈ 参数量 × 0.6GB，再加 1–2GB 运行时开销和 KV cache。12GB 显存跑 14B Q4 很稳；32B 就得 Q3 或部分 offload 到内存，速度掉一个量级。

**第二步：选运行时。** 日常用 Ollama，需要精细控制量化参数时用 llama.cpp，快速试模型用 LM Studio。三者都提供 OpenAI 兼容接口，接 OpenClaw 只需改 provider 的 base URL 和模型名。

**第三步：选模型。** 别只看榜单分数，重点验证 tool calling。我在自己的三个 MCP 工具上跑了一个 20 轮固定脚本（查日程 → 写文件 → 调搜索 → 汇总），14B 级别里 Qwen 系和部分 finetune 版能稳定通过，有些同尺寸模型参数抽取错误率超过 30%。

**第四步：改 context 配置。** Ollama 默认 num_ctx 很小，Agent 一轮多工具调用就截断，而且截断是静默的——表现为 Agent"忘了前文"，极难排查。显式设到 16k–32k，同时算好 KV cache 的显存占用。

**第五步：接入联调。** 先跑单工具调用，再跑多工具链路，最后跑长会话。我把这套流程固化成 benchmark 脚本，换模型或换量化时直接复跑，看 tool call 成功率和延迟。

## 踩坑点

- **静默截断**：context 不够时 Ollama 不报错，直接丢早期上下文。Agent 的诡异行为一半来自这里。
- **tool call 格式差异**：不同模型对 function calling schema 的遵循度差距极大，别信"支持"两个字，用自己的工具实测。
- **长会话内存膨胀**：32k context 跑几轮后 KV cache 能吃掉 3–4GB 显存，再调工具就可能 OOM。Agent 侧要做会话压缩或定期开新会话。
- **量化档位陷阱**：同一模型不同 quant 的表现差距明显，工具调用对量化比对话更敏感，Q4 以下慎用。

## 可复用建议

1. **分层路由**：本地模型接高频、敏感、轻量任务；复杂规划留给云端。按任务类型配不同 provider，隐私和成本都可控。
2. **固化 benchmark**：用真实 MCP 工具链写评测脚本，模型选型从"感觉"变成数据。
3. **锁版本**：模型和运行时都 pin 死版本。本地环境最大的风险不是跑不动，而是"悄悄变了"。

## 总结

本地部署的价值不是替代云端大模型，而是给 Agent 工作流加一层离线、可控、低成本的执行层。把精力花在验证 tool calling 和管好 context 上，14B 级别的本地模型完全能扛起一半的自动化日常。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-06/1bc74ee9144f9ac4.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-06/43eec1b780c2dffc.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-06/5f669ccd015bd8d7.png)

