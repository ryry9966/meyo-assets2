---
title: 一个 Agent 同时接 Telegram 和 Discord：跨平台消息路由实践
feedId: 42509
source: 综合讨论
publishedAt: 2026-10-25
---

## 背景

手上有一个 Telegram 群和一个 Discord 服务器，两边用户都在 @bot 提问。最初的方案是偷懒：起两个 agent 实例，各接一个平台。跑了三周就暴露问题——记忆不同步（一边记过的事另一边不知道）、system prompt 维护了两份且已经漂移、工具和 MCP server 也得配两遍。于是收敛成现在这套方案：**单实例、多通道、统一路由**。

## 问题

拆开其实是三件事：

1. **会话隔离与合并**：Discord 的频道/线程和 Telegram 的 chat 是两套 ID 体系，session key 怎么生成，直接决定"谁和谁共享上下文"。
2. **格式差异**：长度限制（Discord 单条 2000 / TG 4096）、markdown 方言、回复引用方式、媒体附件形态都不同。
3. **节奏差异**：两平台限速模型不同，通知类广播最容易撞限。

## 做法

1. **单实例多通道**。一个 agent 挂两个 channel，workspace、记忆、MCP 工具全部共享，token 只花一份。 secrets 走环境变量，配置示意（字段名以你的版本为准）：

   ```json
   {
     "agents": { "main": { "workspace": "~/claw-workspace" } },
     "channels": {
       "telegram": { "botToken": "${TG_BOT_TOKEN}" },
       "discord":  { "botToken": "${DC_BOT_TOKEN}" }
     }
   }
   ```

2. **先定 session key 策略**。看日志确认 key 形如 `channel:chatId`，即两个平台天然隔离、互不串台。确需跨平台认同一个用户的场景，在 workspace 里维护一份 identity 映射，让 agent 用统一称呼。别一上来就全局合并，串台的排障成本远高于省下的那点配置。

3. **统一信封，晚做适配**。内部消息统一成 envelope（platform / chat / user / text / media），parse_mode、reply 引用、typing 指示这些平台差异全部收敛在适配层，**不进 prompt**。

4. **分块与降级**。出站前按平台限制切块，代码块整体保留不拦腰截断；Discord 超长内容降级为附件文件发送。

5. **媒体立即落地**。入站图片/文件在 ingest 时就下载到本地媒体目录，不存平台 CDN 链接。

6. **出站走队列**。通知类消息先合并、再按各平台限速节流发送。

## 踩坑点

- Discord 后台没开 **Message Content Intent**，bot 收到的 content 全是空字符串，表现为"收到了但不回复"。这是最隐蔽的一个，我第一晚查了两个小时日志。
- Telegram 的 MarkdownV2 转义是无底洞，最终妥协：正文走纯文本加代码围栏，放弃花哨格式，两边观感一致。
- 用 webhook 做跨群同步时忘了过滤 bot 自己的消息，出现回声循环，消息雪崩。加一层 sender 判断解决。
- Discord CDN 链接有时效，历史消息里的图第二天就 404——所以媒体必须落地本地。

## 可复用建议

- **先设计 session key，再写适配器**，顺序反了必返工。
- 平台差异全部下沉到适配层，system prompt 只写一份、只写一份、只写一份。
- 日志统一加 `[tg]` / `[dc]` 前缀，排障时一眼分流。
- 先 dry-run 跑通完整路由和分块逻辑，再放开真实群。

## 总结

一个 Agent 服务两个平台，难点不在接 API，而在会话键设计和格式归一化。把信封结构定好、把平台差异关进适配层，之后想再加 Slack 或飞书，基本就是复制一份 channel 配置的事。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-25/7b4f65665d0d428f.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-25/5c804ff5201a0d19.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-25/843d00a325a725fd.png)

