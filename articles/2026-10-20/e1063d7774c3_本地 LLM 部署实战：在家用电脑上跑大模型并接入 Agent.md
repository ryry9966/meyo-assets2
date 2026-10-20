---
title: 本地 LLM 部署实战：在家用电脑上跑大模型并接入 Agent
feedId: 42114
source: 综合讨论
publishedAt: 2026-10-20
---

# 本地 LLM 部署实战：在家用电脑上跑大模型并接入 Agent

## 背景

对搞 Agent 自动化的人来说，日常 token 消耗大头往往是"量"而不是"难"：MCP 工具调度的循环轮次、日志整理、剪贴板和本地文件的摘要、重复性草稿生成。这些活全走云端 API，一是成本累积，二是剪贴板、内网路径、私有文件这些东西过第三方总归不放心。本地跑一个小模型，定位不是替代云端旗舰模型，而是把这类"高频低难"任务卸载到家里那台机器上。

## 问题

消费级硬件的现实约束摆在那：显存普遍 8–24GB。而且 Agent 场景对模型的要求比聊天严格得多：

- system prompt 加上 MCP 工具列表，轻松吃掉数千 token 的上下文；
- function calling 要求 JSON 结构化输出稳定，小模型这里最容易崩；
- 多轮工具循环里，一次格式漂移就是死循环。

聊天流畅 ≠ 能当 Agent 执行层，这是本地部署最常见的预期偏差。

## 部署步骤

1. **先算显存账**。经验值（Q4_K_M 量化）：7B 约 4.5GB、14B 约 9GB、32B 约 20GB，另留 2–6GB 给 KV cache（随上下文长度增长）。
2. **选推理后端**。Ollama 或 llama.cpp server 起步，两者都提供 OpenAI 兼容端点，OpenClaw 侧改一行 base_url 即可接入；vLLM 留给 24GB+ 且追吞吐的场景。
3. **选模型**。中文 + 工具调用场景，Qwen 系列中小尺寸是稳妥起点。注意：同一模型不同量化档位的 function calling 表现差异明显，选完量化版要实测。
4. **接入 OpenClaw**。模型配置指向本地端点，temperature 压到 0.1–0.3，工具列表做减法——只保留本地模型真正能调动的 3–5 个 MCP 工具。
5. **真实任务压测**。用自己的自动化脚本连跑 20 轮，记录成功率、tokens/s、工具调用 JSON 合法率，比任何通用 benchmark 都有参考价值。

## 踩坑记录

- **上下文长度是显存黑洞**。8GB 卡上把 context 开到 32k 直接 OOM；KV cache 量化到 q8_0 能省近半显存，但长文召回质量会掉一点。
- **量化档位杀 function calling**。Q2/Q3 档的 JSON schema 遵循能力基本不可用，参数瞎编；14B 以下坚持 Q4_K_M 或更高。
- **最慢的不是显存不够，是溢出到内存**。部分层 offload 到内存后，速度能从 40 tok/s 掉到 2 tok/s。启动日志里确认层数全部落在 GPU 上。
- **高温让 Agent 打转**。temperature 高于 0.6，小模型容易反复调用同一工具或格式漂移——这是温度敏感，不是模型坏了。

## 可复用建议

- **选型基准**（Q4_K_M）：8GB 显存 → 7–8B；12GB → 14B；16GB → 14B 大上下文，或试 Qwen3-30B-A3B 这类激活参数小的 MoE，日用体验意外地好；24GB → 32B。
- **用 OpenAI 兼容端点做抽象层**，云端/本地切换只改配置；"本地起草、云端定稿"的混合路由比纯本地实用得多。
- **记录每次 Agent 运行的 token 消耗**，上下文膨胀多半来自冗余的工具描述，定期给 MCP 工具列表瘦身。
- **锁模型版本**（记录 digest），评测脚本和配置进同一个仓库，半年后还能复现今天的结果。

## 总结

本地 LLM 的现实定位：结构化抽取、日志整理、简单工具调度、隐私敏感文本，这些高频且难度可控的任务；复杂推理和长链规划仍交给云端。一台 16GB 显存的机器，Qwen 14B Q4 + OpenClaw + 精简过的 MCP 工具集，就是一套稳定的日用组合。先跑真实任务再定配置，别为 benchmark 数字交学费。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-20/63f6e38e1ca8107c.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-20/1e1ee12c6e59879c.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-20/ba5e2b000d2c82b6.png)

