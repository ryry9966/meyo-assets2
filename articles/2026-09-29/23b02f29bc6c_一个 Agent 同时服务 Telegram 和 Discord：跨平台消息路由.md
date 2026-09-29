---
title: 一个 Agent 同时服务 Telegram 和 Discord：跨平台消息路由实践
feedId: 39511
source: 综合讨论
publishedAt: 2026-09-29
---

## 背景

社区用户同时分布在 Telegram 和 Discord，两边都在和同一个 Agent 对话。最早的做法是各跑一个 bot、各自维护会话，很快暴露三个问题：prompt 和 MCP 工具配置要维护两份；两边的记忆完全割裂，A 平台说过的事 B 平台不知道；排障要看两套日志。于是改成「多通道、单核心」：两个平台只做传输适配，Agent 核心（会话存储 + MCP 工具）只有一份。

## 问题

难点不在跑通，而在平台差异由谁消化：

- 消息模型不同：mention、实体、附件、线程语义各一套；
- 输出限制不同：Telegram 单条 4096 字符，Discord 2000；
- Markdown 方言不同，转义规则互相打架；
- 限流策略不同，出站必须独立节流。

## 做法与步骤

**1. 先定义归一化消息模型**，这是整个方案的地基：

```json
{
  "platform": "telegram | discord",
  "chat_id": "...", "user_id": "...",
  "message_id": "...", "thread_id": null,
  "text": "...",
  "mentions": [], "reply_to": null,
  "attachments": [{ "type": "image", "url": "..." }]
}
```

**2. 每个 adapter 只做两件事**：inbound 把平台事件转成归一化模型；outbound 把核心回复渲染成平台格式。Telegram 走 webhook（或 getUpdates + offset），Discord 走 gateway 并开启 message content intent。

**3. 路由与会话**：session key 用 `(platform, chat_id)`；群聊默认只在被 @ 或被回复时响应，避免刷屏。会话状态统一存一份（SQLite 就够），核心不感知平台。

**4. 出站统一过队列**：按 chat 维度排队，Discord 按每通道 5 条/5 秒、Telegram 按每聊天约 1 条/秒做令牌桶；超长消息先按代码块围栏切，再按平台上限二次切分。

**5. 幂等**：处理前查 `(platform, message_id)` 去重表，重连重推直接跳过。

## 踩坑点

- **Markdown 转义最耗时**。Telegram MarkdownV2 和 Discord 的转义字符集不同，代码块内的处理方式也不一样。后来放弃「统一 markdown」，核心只输出纯文本 + 结构化代码块，由各 renderer 自行渲染，正则噩梦少了一半。
- **按字数硬切会劈开代码块**。必须先扫描围栏位置再切。
- **重复投递**。Discord 断线重连后事件重放、Telegram getUpdates 超时后 offset 回退，都会重发消息；没有幂等层就会重复回答。
- **媒体格式**。Telegram 语音是 ogg/opus，Discord 侧要转码；贴纸和 GIF 建议一期降级为占位符，别追求全兼容。
- **身份别急着打通**。同一人在两个平台就是两个 session，先保持隔离；之后再做显式绑定（比如绑定码），隐式合并极易串上下文。

## 可复用建议

- adapter 保持薄，业务逻辑全部进核心，新增平台 = 实现两个函数；
- 出站永远过队列，不要在 handler 里直接发消息；
- 日志统一带 `platform/chat_id/message_id`，一次跨平台链路能串起来；
- 准备 dry-run 模式：inbound 正常、outbound 只打日志，改 renderer 时的回归全靠它。

## 总结

跨平台消息路由的本质，是把平台差异压缩进两个薄 adapter，让 Agent 核心只面对一个稳定的归一化模型。做完后最直观的收益：配置一份、记忆一份、日志一份，新增通道的成本降到两个函数。剩余的问题基本都出在细节——转义、限流、幂等。建议第一天就把去重表和出站队列建好，比事后补救便宜得多。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-29/97bf3a71b5de1c9d.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-29/8093fe7423445e1f.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-29/ea981d75d94d20a7.png)

