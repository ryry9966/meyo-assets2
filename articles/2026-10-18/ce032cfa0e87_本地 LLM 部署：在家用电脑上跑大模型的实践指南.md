---
title: 本地 LLM 部署：在家用电脑上跑大模型的实践指南
feedId: 41927
source: 综合讨论
publishedAt: 2026-10-18
---

## 背景

玩 Agent 和自动化的人迟早会算一笔账：OpenClaw 这类常驻 agent 加上频繁的 MCP 工具调用，走云端 API 按量计费，长期跑下来不便宜。而且自动化脚本要读私人消息、本地文件，数据不出门是有真实需求的场景。量化技术成熟之后，7B–32B 级别的模型在消费级硬件上已经能跑，值得认真试一次。

## 问题

但"能跑"和"能用"是两回事。Agent 场景对模型的要求远高于聊天：

- 必须稳定输出结构化 tool call，小模型经常格式崩坏
- 系统 prompt + 一堆 MCP 工具的 schema，轻松吃掉几千 token
- KV cache 的显存开销普遍被低估

## 做法

### 1. 先算显存账

经验值（Q4 量化）：7B/8B 约 5–6 GB，8G 显卡或 16G 内存 Mac 可跑；14B 约 9–11 GB；32B 约 18–20 GB，24G 卡起步。注意预留 KV cache：上下文开到 16k–32k 还要额外吃 1–4 GB。

### 2. 选运行时

- **Ollama**：最省事，自带 OpenAI 兼容接口，适合快速验证
- **llama.cpp**：可控性最强，参数能精细调
- **vLLM**：吞吐高，但基本要 Linux + NVIDIA，家用机一般用不上

### 3. 选模型，优先看 tool call

第一筛选条件不是榜单分数，而是 function calling 稳定性。Qwen 系列、Llama 3.1+ 支持较好。量化版本选官方或社区高下载量的，别随手下不明来源的 GGUF——tool 模板被改坏的量化很常见。

### 4. 接入 OpenClaw

把 provider 指到本地 OpenAI 兼容端点（如 `http://localhost:11434/v1`），两件事必做：显式调大上下文（Ollama 默认 num_ctx 很小）；keep_alive 设长，避免每轮请求重新加载模型。然后用一个最简单的 MCP 工具（比如本地文件读写）做冒烟测试。

### 5. 用自己的任务测，别看跑分

固定 10–20 条你真实的 agent 任务（多步调用、失败重试、长输出），记录成功率和端到端延迟。榜单分数和你的场景经常不相关。

## 踩坑点

1. **num_ctx 没调**：跑到一半模型"失忆"，工具 schema 被截断，表现为莫名不调用工具
2. **KV cache 超预期**：权重放得下，上下文一长就 OOM
3. **别用 Q2/Q3 量化跑 agent**：tool call 格式错误率明显上升
4. **首 token 延迟**：系统 prompt 很长时，消费级显卡处理 prompt 慢，agent 每轮重喂一遍，体感卡顿
5. **简单测试通过 ≠ 多步链路可用**，务必端到端验证

## 可复用建议

- 把"模型 + 量化 + 运行时版本"当依赖钉死，记进仓库，升级前跑回归
- 混合路由：复杂规划走云端，分类/抽取/格式化走本地，平衡成本和体验
- 全程走 OpenAI 兼容接口，后端可随时在 Ollama / llama.cpp / 云端之间切换
- macOS 用户别把内存上限拉满，给系统留余量，swap 一上速度就崩

## 总结

家用机跑本地 LLM 做 agent 后端是成立的，前提是把它当工程问题对待：算好显存账、选 tool call 友好的模型和量化、显式配上下文、用自己的任务建回归集。我 16G 内存的 Mac 跑 8B Q4 处理日常自动化够用，重规划仍交给云端。本地不是替代，是分层路由里的一层。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-18/593f1af1fa35dd68.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-18/09ef4ba2e9a61111.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-18/9a16d4c4aae47884.png)

