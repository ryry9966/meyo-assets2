---
title: 一个 Agent 服务两个平台：Telegram + Discord 消息路由实践
feedId: 38316
source: 综合讨论
publishedAt: 2026-09-21
---

## 背景

OpenClaw 的频道适配层天然支持多渠道接入。我之前只给 Agent 接了 Telegram，后来常用的社区迁到了 Discord，于是想把同一个 Agent 同时挂到两边——目标很朴素：**一个大脑、两个入口、行为一致、维护成本不翻倍**。不搞两套 bot、两份 prompt，那基本等于养两个性格逐渐分叉的员工。

## 问题在哪

动手前明确了几类真实问题：

1. **身份碎片化**：同一个人在 Telegram 和 Discord 是两个不同的 ID，Agent 无法自然识别"这是同一个人"。
2. **会话隔离**：两渠道共享同一个上下文会互相泄露；完全不共享则用户跨平台切换时丢上下文。
3. **格式差异**：Telegram 的 MarkdownV2 和 Discord 的 Markdown 语法不同；单条上限一个是 4096 字符，一个是 2000。
4. **回环与限速**：bot 互相 @ 导致死循环，双渠道同时推送时触发 429。

## 做法

1. **同时启用两个 channel**，各自填 token：

```json5
{
  "channels": {
    "telegram": { "botToken": "..." },
    "discord":  { "botToken": "..." }
  }
}
```

2. **会话按 `channel:chatId` 隔离**，再拆一层"长期记忆"：用户偏好、项目上下文放共享 workspace，会话级历史留在各自 session。这样跨平台不丢人格，但不会串聊天记录。
3. **身份映射手动维护**：一个简单的映射文件，绑定 Telegram 用户 ID ↔ Discord 用户 ID。不要让 Agent 自动猜合并，错了很难回滚。
4. **出站统一过 formatter**：Agent 只输出内部统一格式，发送前按渠道做格式化——Discord 侧截断到 2000，Telegram 侧做 MarkdownV2 转义。
5. **路由规则数据化**：告警类走 Telegram 私聊、社区答疑走 Discord 频道，这些写成配置，不写死在 prompt 里。

## 踩坑点

- **MarkdownV2 转义**：漏一个 `_` `*` `(` 就是 400 错误。最后干脆让 Agent 输出接近纯文本，富文本交给 formatter 统一处理，比两边各自渲染稳定一个量级。
- **Discord 截断**：长代码回复直接超限，且不能把代码块截一半。策略是按段落切分，大段代码降级为"存文件 + 发摘要"。
- **429 限速**：双渠道群发时 Discord 先炸。加一个全局出站队列 + 指数退避，比各渠道各自重试可靠。
- **回环**：Discord 里别的 bot 会 @ 我们的 bot。入站 handler 里直接丢弃所有 bot 来源消息（Telegram 看 `is_bot`，Discord 看用户标记），一行代码省一晚上排障。
- **定时任务**：心跳只跑一份，别挂在某个 channel 的 session 里，否则该渠道重连会重复触发。

## 可复用建议

- 渠道差异全部收敛到**入站归一化 + 出站格式化**两个薄层，Agent 核心只处理统一的内部消息结构。
- 身份映射宁可手动维护，也别自动合并。
- 跨平台行为的开关放配置，prompt 里只留策略描述。
- 日志带 channel 标签，排障时一眼分清消息来源。

## 总结

接通后实际维护成本没有翻倍，增量主要在 formatter 和身份映射两小块。OpenClaw 的适配层把传输差异挡在 Agent 之外，剩下的就是工程老三样：归一化、限速、去重。后续要加 Slack 或飞书，预计只需新增一个 adapter 和对应 formatter，核心不用动。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-21/10795881c606eb33.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-21/d56abf2c6ecfd8a0.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-21/bd287acef7901915.png)

