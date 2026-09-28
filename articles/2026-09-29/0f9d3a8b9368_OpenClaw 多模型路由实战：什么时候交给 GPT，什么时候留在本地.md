---
title: OpenClaw 多模型路由实战：什么时候交给 GPT，什么时候留在本地
feedId: 39396
source: 综合讨论
publishedAt: 2026-09-29
---

## 背景

OpenClaw 的模型层是开放的：`openclaw.json` 里可以同时挂多家云端 provider，也能接任何 OpenAI 兼容端点——Ollama、LM Studio、vLLM 都行。但社区里最常见的用法是全链路一把梭：主 agent、子 agent、心跳任务统统打旗舰模型。跑两周看账单，再看延迟曲线和内网数据流向，问题就浮出来了。

## 问题

拆开是三件事：

1. **成本错配**：心跳、消息摘要、格式清洗这类高频小任务用了最贵的模型，token 占比六成以上，质量收益约等于零。
2. **延迟不稳**：简单指令也要等旗舰的首 token，交互手感被拖垮。
3. **隐私边界模糊**：日程、笔记、内网文档混在上下文里出境，事后才意识到。

根因不是"选哪个模型"，而是缺一条明确的分工边界。

## 做法与步骤

**1. 先定档位**。旗舰：复杂规划、多工具编排、长上下文推理；中档云端：日常对话与改写；本地小模型：摘要、分类、结构化提取——高频、可容错、数据敏感的活。

**2. 按 agent 绑模型，别按全局绑**。OpenClaw 允许每个 agent 单独指定 primary 和 fallbacks，这是做路由最自然的抓手：

```jsonc
{
  "models": {
    "providers": {
      "openai": { "models": ["gpt-4.1", "gpt-4.1-mini"] },
      "local": {
        "baseUrl": "http://127.0.0.1:11434/v1",
        "api": "openai-completions",
        "models": ["qwen3:14b"]
      }
    }
  },
  "agents": {
    "defaults": {
      "model": {
        "primary": "openai/gpt-4.1",
        "fallbacks": ["openai/gpt-4.1-mini", "local/qwen3:14b"]
      }
    }
  }
}
```

主对话 agent 用旗舰保质量；另建一个后台子 agent，primary 直接指到本地模型，负责转写稿摘要、链接归档、定时巡检。

**3. fallback 是保命链，不是省钱链**。旗舰 → 中档 → 本地是降级兜底，防 API 限流时整个 agent 瘫痪。想"先小后大、复杂再升级"，得自己加一层分类——一个挂了路由提示词的小 agent 或插件 hook——OpenClaw 不会替你判断任务复杂度。

**4. 加观测**。按周拉 provider 用量和延迟分布，数据是路由策略唯一的裁判。

## 踩坑点

- **本地模型 tool calling 是重灾区**：14B 以下参数的工具调用格式经常崩，agent 反复调用或乱填参数。接入 agent 流程前先单独压测。
- **context window 静默截断**：本地小模型 8k 窗口，长会话历史塞进去直接砍头，表现为"突然失忆"，很难第一时间归因到模型切换。
- **中文能力断崖**：同级参数下英文指令遵循尚可、中文长文总结明显下滑。中文为主的任务，本地档建议 14B 起步并实测。
- **fallback 反向烧钱**：本地端点没起来，请求静默落到旗舰，账单不降反升。本地 provider 加健康检查，挂了就报警而不是悄悄降级。

## 可复用建议

一张最小任务矩阵，照抄即可：

| 任务类型 | 档位 | 理由 |
|---|---|---|
| 规划、编排、排障 | 旗舰 | 错一次的代价远超省下的钱 |
| 日常对话、改写 | 中档云端 | 质量差距小，成本砍半 |
| 摘要、分类、提取 | 本地 | 高频、可容错、敏感数据 |
| 心跳、定时巡检 | 本地 | 纯烧钱区，务必下沉 |

再补三条经验：路由策略一次只动一档，观察一周再迭代；本地模型选原生支持 tool calling 的系列；用量报表里单独盯 fallback 触发次数，它是配置健康度最灵敏的信号。

## 总结

多模型路由不是省钱技巧，而是把质量、成本、延迟、隐私四个约束显式化的工程决策。OpenClaw 的 per-agent 绑定加 fallback 链已经覆盖大多数场景，剩下的活是拿数据把边界画准。先用观测说话，再动配置——顺序别反。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-29/b9cf3af4f04f66af.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-29/0a0e556edb7ab73d.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-29/a346ac99f865bbb5.png)

