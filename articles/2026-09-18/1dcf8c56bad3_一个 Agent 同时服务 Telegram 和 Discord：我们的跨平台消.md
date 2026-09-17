---
title: 一个 Agent 同时服务 Telegram 和 Discord：我们的跨平台消息路由实践
feedId: 38004
source: 综合讨论
publishedAt: 2026-09-18
---

## 背景

我们的用户一半在 Telegram，一半在 Discord。最早的做法是跑两个 Agent 实例各接一个平台，三个月后两边的提示词、工具配置、知识库开始漂移：Telegram 侧修掉的 bug，Discord 侧忘了同步。于是决定收敛为一个 Agent，平台只做入口。

## 问题拆解

把两个平台接进同一个 Agent，表面上是接两个 SDK，实际是四类工程问题：

1. **格式方言**：Telegram 的 MarkdownV2 转义规则和 Discord 的 Markdown 完全不同，代码块、链接、粗体要分开处理。
2. **身份与会话**：同一用户在两边是两个 ID，会话若直接按 `platform:user_id` 做键，跨平台上下文天然割裂。
3. **限流与长度**：Discord 单条 2000 字符、按 channel 限流；Telegram 单条 4096 字符、全局限流更紧。
4. **部署形态**：Telegram 走 long polling，Discord 走 gateway WebSocket，断线重连逻辑各自独立。

## 做法：适配层薄，核心层厚

架构上只有一个原则：**平台差异全部关在 adapter 里，Agent 核心只认识一种内部消息格式**。

```
telegram adapter ──┐                                 ┌─> formatter(tg) ─> telegram
                   ├─> 统一消息总线 ──> Agent 核心 ──┤
discord adapter ──┘                                 └─> formatter(dc) ─> discord
```

1. **定义规范消息**：`{ sender, session_key, text, attachments, platform }`，adapter 负责把原生事件翻译成它。
2. **会话键设计**：默认 `platform:user_id`，显式绑定的用户通过 `/link` 配对码合并到同一逻辑会话。
3. **出站格式化**：核心输出标准 Markdown，formatter 按目标平台做转义和裁剪，超长消息按代码块边界切片，而不是按字符数硬切。
4. **发送队列**：每个平台一个带令牌桶的出站队列，429 / flood wait 时退避重试，重试期间消息去重。

在 OpenClaw 里打开 telegram、discord 两个 channel 本身不难，真正的工作量在第 1、3 步。建议先验证框架默认 formatter 对你的内容形态（长代码块、表格）够不够用，不够再替换。

## 踩坑点

- **MarkdownV2 转义**是第一大坑：漏转一个符号，整条消息静默 parse error。我们的策略是能不用就不用，代码块场景才切。
- **切片破坏代码块**：按字符数硬切会把代码块切成两半，后半段渲染成乱码；切片器必须感知围栏状态。
- **别做自动身份合并**：最初想用用户名匹配两边身份，重名用户直接串了会话；改成显式配对码之后才稳定。
- **长任务反馈**：工具链跑超过 30 秒时，Discord 可以用 typing 状态，Telegram 更适合先回一条"处理中"再编辑。这个交互差异要写进 adapter，别泄漏到核心逻辑。
- **重连幂等**：Discord gateway 重连后可能重放事件，Telegram 的 getUpdates 有 offset。两边都要做事件去重，否则重启一次 Agent 会把同一批消息再答一遍。

## 可复用建议

1. **fixture 回放测试**：把两个平台的原始报文各存十几条作为测试集，adapter 改动跑一遍回放，比手点快得多。
2. **分平台打点**：入站速率、出站失败数、formatter 报错数分开统计，出问题立刻能分清是平台侧还是核心侧。
3. **新平台即新 adapter**：收敛架构后接第三个平台只花了一天，验证了"适配层薄"这个约束值得守住。
4. **能力降级显式化**：目标平台不支持的能力（比如 thread）在 formatter 里显式降级，不要抛错。

## 总结

跨平台路由的核心不是"接两个 SDK"，而是把平台差异压缩进两个薄 adapter，让 Agent 核心面对唯一的内部消息协议。做完之后最大的收益不是省了一台机器，而是提示词、工具、知识库只维护一份，功能迭代不再需要两边对齐。如果你的用户也分散在多个平台，建议尽早收敛——漂移的代价会随时间指数增长。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-18/44a286779c714411.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-18/9d1ce65693127336.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-18/48b222f7f3f5aaa6.png)

