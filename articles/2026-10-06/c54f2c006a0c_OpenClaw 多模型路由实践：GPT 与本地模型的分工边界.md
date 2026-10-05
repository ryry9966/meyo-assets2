---
title: OpenClaw 多模型路由实践：GPT 与本地模型的分工边界
feedId: 40600
source: 综合讨论
publishedAt: 2026-10-06
---

## 背景

OpenClaw 的默认用法是把所有会话压到一个主力模型上。跑一段时间通常撞上两个极端：要么账单失控——定时任务、频道摘要、无人值守的自动化全在烧旗舰模型；要么一刀切全换本地小模型——工具调用频繁失败，Agent 表现断崖式下滑。多模型路由要回答的不是"哪个模型最强"，而是"哪类请求配哪类模型"。

## 问题拆解

先分类，再谈路由，建议按三个维度：

- **敏感性**：读本地文件、解析聊天记录、接触密钥的任务，数据尽量不出机器；
- **能力要求**：多步工具调用、长链路规划对模型要求高；摘要、打标签、格式转换属于低风险任务；
- **频率与成本**：一分钟一次的 cron 和一天问三次的交互，预算逻辑完全不同。

## 做法与步骤

1. 在配置里同时挂两组 provider：云端（OpenAI 兼容接口）和本地（Ollama / LM Studio 的 OpenAI 兼容端点）；
2. 主 Agent 用旗舰云端模型，保证工具调用和规划的稳定性，同时把本地模型写进 `fallbacks`，云端抖动时至少不掉线；
3. 把高频、低风险的自动化拆成独立子 Agent（如摘要、日报），显式指定本地小模型——这部分往往占调用量大头，路由收益最明显；
4. 留一个手动逃生口：单次会话需要强推理时用 `/model` 临时切旗舰模型，用完切回。

配置大致长这样（节选，字段名以你所用版本的 schema 为准）：

```jsonc
{
  "models": { "providers": {
    "cloud-gpt":  { "baseUrl": "https://api.openai.com/v1", "api": "openai-completions", "apiKey": "sk-..." },
    "local-qwen": { "baseUrl": "http://127.0.0.1:11434/v1", "apiKey": "ollama" }
  }},
  "agents": {
    "defaults": { "model": { "primary": "cloud-gpt/gpt-4.1",
                             "fallbacks": ["local-qwen/qwen3:14b"] } },
    "list": [
      { "id": "main",   "default": true },
      { "id": "digest", "model": { "primary": "local-qwen/qwen3:14b" } }
    ]
  }
}
```

## 踩坑点

- **小模型别进工具链**。本地小模型聊天没问题，但 function calling 的格式遵循很差，MCP 工具一多就开始幻觉参数。上路由前用真实工具链跑一轮回归，别只测聊天。
- **上下文窗口**。本地模型常见 8k–32k，而 OpenClaw 会话会持续膨胀。给跑本地模型的 Agent 设会话重置或上下文裁剪，否则一半请求会静默截断。
- **fallback 的静默降级**。云端报错自动落到本地很方便，但事后看不出答案变差了。务必在日志里记录每轮实际服务的模型，出问题能对账。
- **cron 是成本黑洞**。旗舰模型跑每分钟级定时任务，一天账单够心疼很久；定时任务优先路由到本地或便宜模型。

## 可复用建议

- 路由顺序：先按敏感性分流，再按能力要求，最后才看成本；
- 每条路由都留日志：模型 ID、token 数、延迟，月底据此调一次任务映射表；
- 量化档位保守选：代码任务用 Q4 量化小模型，出错率明显抬升，宁可上大一档；
- 本地端点统一走 OpenAI 兼容协议，换底层推理引擎（Ollama ↔ LM Studio）不用动 OpenClaw 配置。

## 总结

多模型路由不是"本地替代云端"，而是按任务分配资源：敏感和高频的给本地，复杂和低频的给云端，中间靠 fallback 和日志兜底。OpenClaw 的多 provider 加 per-agent 覆盖机制足够支撑这套玩法，关键是先花半天把任务分类表列出来——这一步比任何调参都值钱。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-06/e1d47e947ca72905.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-06/531a9717ac13b676.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-06/204b0346233edf3d.png)

