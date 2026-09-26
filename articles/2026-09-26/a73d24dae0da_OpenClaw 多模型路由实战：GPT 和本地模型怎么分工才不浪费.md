---
title: OpenClaw 多模型路由实战：GPT 和本地模型怎么分工才不浪费
feedId: 39069
source: 综合讨论
publishedAt: 2026-09-26
---

## 背景

我的 OpenClaw 网关跑了半年多，接了 Telegram 和桌面端，日常几百条消息：日程整理、网页摘要、写点脚本、偶尔翻相册找照片。头两个月所有请求都打在云端旗舰模型上，月底一看账单不便宜；与此同时家里那块 24G 显存的卡大部分时间闲着。多模型路由要解决的从来不是"能不能同时接两个 provider"——OpenClaw 本身就支持——而是**每个任务该给谁**。

## 问题

三个诉求在打架：质量、成本、隐私。全走云端，贵，而且相册、日记、服务器日志这类数据必须出网；全走本地，14B 以下的模型在多步 agent 循环和复杂工具调用上确实不稳。最后我落到一个混合方案：按任务属性路由，而不是按心情切。

## 做法

**1. 先分层，再谈路由。** 把任务分三类：A 类重推理（规划、写代码、多步 agent 循环）走云端；B 类轻任务（摘要、翻译、格式化、通知文案）走本地 7B~14B；C 类敏感数据（相册、日记、日志）无论难度强制本地。

**2. 注册本地 provider 并配 fallback。** llama.cpp server 或 Ollama 的 OpenAI 兼容端点直接挂进 `openclaw.json`，主模型设 GPT 级，fallback 链放本地模型，云端限流时至少不至于整体不可用：

```json
{
  "models": {
    "default": "openai/gpt-4.1",
    "fallbacks": ["local/qwen2.5-14b-instruct"]
  },
  "skills": {
    "photo-journal": { "model": "local/qwen2.5-14b-instruct" }
  }
}
```

**3. skill 级覆盖 model。** 读相册、整理日记这类 skill 固定到本地模型，其余走默认。显式声明比依赖全局默认可靠得多。

**4. 路由规则保持蠢。** 消息长度阈值、命中关键词（"总结""翻译"）、特定 channel 直接判本地，否则走默认。不要让"路由判断"本身再调一次大模型——那是用钱买路由。

**5. 落观测。** 网关日志带 model 字段，按天统计 token 消耗、P95 延迟、fallback 次数。没有这三条曲线，路由策略是调不下去的。

## 踩坑点

- **tool call 兼容性**：部分量化模型会输出损坏的 JSON，表现为插件"随机失灵"。本地模型只跑不依赖复杂工具链的 skill，工具深度限 1。
- **量化等级**：Q4 以下在长 system prompt（OpenClaw 注入的上下文不小）下退化明显。别信跑分，拿自己的真实 prompt 测。
- **显存抢占**：本地推理和转写、绘图服务抢卡，不设并发上限的话路由"成功"但首 token 等你 30 秒。
- **静默降级**：fallback 不打日志，某周回答质量整体下滑你都不知道原因。fallback 必须落日志，最好带告警。

## 可复用建议

- 第一分类轴用"数据能不能出网"，而不是"任务难不难"。隐私线画清楚后，剩下的路由问题都简单。
- 每个 skill 显式写 model，半年后配置还能看懂。
- 每周人工抽查 20 条本地模型的回答，比任何 benchmark 都真实。
- 给自己设月度 token 预算，超限先砍的是被误路由到云端的 B 类任务。

## 总结

多模型路由的价值不在省那点 API 费，而在于把隐私边界和可用性从"习惯"变成"配置"。先分层、再规则，最后才考虑更聪明的路由。OpenClaw 现有的 skill 级 model 覆盖加 fallback 链已经够用，别急着给路由本身再养一个模型。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-26/9275b178e648afc8.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-26/dff2ca480819d953.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-26/1a543688bbbd4eaa.png)

