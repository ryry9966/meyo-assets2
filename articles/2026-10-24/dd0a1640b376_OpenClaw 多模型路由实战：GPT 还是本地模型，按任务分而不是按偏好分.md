---
title: OpenClaw 多模型路由实战：GPT 还是本地模型，按任务分而不是按偏好分
feedId: 42415
source: 综合讨论
publishedAt: 2026-10-24
---

## 背景

OpenClaw 的默认用法往往是“一个模型走天下”——要么全走云端旗舰，要么为了省钱全压到本地小模型。自动化任务跑上几周，两边的短板都会暴露：云端账单涨得快，本地模型在多步工具调用上掉链子。

## 问题

实际负载是高度异质的：

- 消息分类、摘要、定时提醒：量大、单次 token 少、对推理要求低；
- 多步 Agent 任务、代码修改、MCP 工具编排：量小、单次贵、容错率低；
- 涉及本地文件、私有笔记、内网数据的内容：不适合出网。

路由的目标不是“哪个模型强”，而是给每类任务匹配“够用且便宜”的模型。

## 做法

1. **先给任务打标签。** 在 skill/插件的描述里写清任务类型，路由才有依据。
2. **配好 provider 池：** 云端主力（推理强）+ 本地 Ollama/vLLM 小模型（快、免费、隐私）+ fallback 链。
3. **规则从粗到细：** 是否涉及隐私/本地数据 → 强制本地；是否多步工具调用/代码 → 云端旗舰；简单分类摘要 → 本地小模型。
4. **用子 agent 分流：** 主 agent 用强模型做规划和审查，批量子任务（逐条翻译、批量打标）下发给本地模型。
5. **打点观测：** 记录每次调用的模型、token 数、耗时、是否触发 fallback，跑一周看分布再调规则。

伪配置示意：

```yaml
routing:
  rules:
    - match: { privacy: true }      -> local
    - match: { tools: multi-step }  -> cloud-flagship
    - match: { task: classify }     -> local-small
  fallback: local-small -> cloud-flagship
```

## 踩坑点

- **本地小模型的 function calling 不稳：** 格式对但参数错，表现为“工具静默失败”。凡要动 MCP 工具的，别省这个钱。
- **上下文窗口差异：** 8k 本地模型接了 32k 会话历史，截断后丢关键约束，路由层要带长度判断。
- **fallback 是隐性成本：** 本地失败静默转云端，月底账单多 40%。fallback 必须打点告警。
- **嵌入模型不匹配：** 知识库云端嵌入写入、本地嵌入查询，检索质量直接崩，读写必须同源。
- **中文 token 效率差异大：** 按英文 benchmark 估本地模型的成本会明显失真。

## 可复用建议

- 做一张“任务 × 模型”决策表放在配置旁，新任务先对表再接。
- 每条路由规则配 20 条左右的小评测集，换模型前跑一遍。
- 预算护栏放在路由层而非账单层：单日 token 超阈值自动降级到本地。
- 模型配置带版本号进 git，回滚有据可查。

## 总结

多模型路由的价值不在“用了几个模型”，而在于你对自己的负载做过分类。在 OpenClaw 里把路由规则写显式、把 fallback 打上点、把评测集建起来，云端和本地就能各干各的活：强模型管推理，本地模型管吞吐和隐私。先跑一周调用统计，再动手改配置，比凭感觉调可靠得多。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-24/f73ecf79d07d0986.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-24/d6736e4e6b7fef13.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-24/c0260c89f6531e35.png)

