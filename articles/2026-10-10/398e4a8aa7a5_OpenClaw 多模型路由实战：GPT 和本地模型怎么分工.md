---
title: OpenClaw 多模型路由实战：GPT 和本地模型怎么分工
feedId: 41094
source: 综合讨论
publishedAt: 2026-10-10
---

## 背景

OpenClaw 默认用一个 primary 模型跑所有事：闲聊、MCP 工具调用、heartbeat 巡检、定时摘要，全走同一条路。接上 Ollama 之后，很多人会走向两个极端：要么全上云端（账单和隐私焦虑），要么全本地（小模型把 agent 的工具调用搞得一团糟）。这篇分享一套我们实际跑过的分工方案。

## 问题：不是“哪个模型强”，而是“哪个任务配哪个模型”

把工作负载拆开看，特征差异很大：

- **高频短任务**：消息分类、群聊摘要、heartbeat、闲聊回复——量大、单次 token 少、出错代价低。
- **重 agent 任务**：多步工具编排、MCP 链式调用、长上下文（整份文档或代码库）——频率低，但对推理能力和工具调用格式稳定性要求极高。
- **隐私敏感任务**：涉及私人消息、密钥、家庭数据，不希望出网。

一个模型通吃的结果，要么是钱白花，要么是重任务被搞砸。

## 做法：分 agent + 分渠道 + 兜底链

1. **接本地 provider**。在 `openclaw.json` 的 `models.providers` 里加 Ollama（OpenAI 兼容端点，`baseUrl` 指向 `http://127.0.0.1:11434/v1`），先确认模型列表能正常拉到。
2. **配 primary + fallbacks**。主力 agent 用云端旗舰模型，fallback 链从云端次旗舰排到本地大参数模型——本地只用于断网和限流兜底，不作为日常。
3. **拆一个轻量 agent**。单独建“日常杂务”agent，primary 直接指向本地模型（Qwen 32B 级别够用），只挂少量工具；把 heartbeat、群聊摘要绑到它身上。
4. **按渠道分流**。私人日常频道走轻量 agent；需要写代码、跑长链路的会话走主力 agent。会话内可以用 `/model` 临时切换，干完重活再切回。
5. **记账再调**。用 `/status` 观察各会话的模型、上下文占用和成本，跑一周再调整分工比例，别第一天就定死。

## 踩坑点

- **小模型工具调用不稳**：8B 级别模型生成工具参数的 JSON 经常格式错，agent 陷入重试循环。结论：工具多的 agent 别用小模型当 primary。
- **上下文窗口不对等**：本地 32k 窗口塞进 OpenClaw 的 system prompt 和工具描述后所剩无几，长会话提前截断。
- **embedding 换模型要重建索引**：memory 检索换了 embedding 模型，旧向量全部失配，召回质量骤降，必须重跑索引。
- **静默 failover 破坏连续性**：fallback 触发后同一会话换了“脑子”，长对话风格会突变，重要会话建议 pin 住模型。
- **VRAM 争抢**：本地同时跑语音转写和 LLM 容易 OOM，给 Ollama 设并发和显存上限。

## 可复用建议

- 判断只记三问：**频率高吗？能出网吗？要稳定的工具调用吗？** 高频、不出网、不重工具 → 本地；反之云端。
- 本地模型当“专用工”，不当“备胎大脑”：绑定单一轻量 agent，职责越窄越稳。
- fallback 链从强到弱排，本地放最后——兜底优先于省钱。
- 每次换模型，先在测试会话跑一遍典型工具链，再切到生产频道。

## 总结

多模型路由的价值不在“省了多少钱”，而在让任务特征和模型能力对齐。OpenClaw 目前靠 agent 拆分、渠道绑定和 fallback 链实现路由，机制简单、行为可预期。先给任务分类，再动配置，比追任何“最强模型”都有效。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-10/fce42611cb5e525e.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-10/368aeb6f024c23fc.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-10/e105a5787c78c943.png)

