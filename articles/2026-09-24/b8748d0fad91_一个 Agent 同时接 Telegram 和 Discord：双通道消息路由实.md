---
title: 一个 Agent 同时接 Telegram 和 Discord：双通道消息路由实践
feedId: 38683
source: 综合讨论
publishedAt: 2026-09-24
---

## 背景

我们的用户一半在 Telegram 群，一半在 Discord 服务器。最初的方案是跑两个 bot、两份配置，结果不到两周就崩了：两边记忆不同步，同一个 agent 在 TG 和 DC 里像两个人格，运维上改一处漏一处。目标很明确——**单 gateway、单 agent 工作区、双通道接入**。

## 问题拆解

真正要解决的不是"能不能连上"，而是四件事：

1. **会话隔离与共享**：私聊和频道是不同上下文，但长期记忆希望共享；
2. **格式差异**：Discord 走 Markdown/embed，Telegram 用 HTML 转义，规则完全不同；
3. **触发语义**：Discord 频道不能每条都回，需要 mention/reply 触发；
4. **限流模型**：两边速率限制不一样，长回复策略要分开。

## 做法

**第一步：单 gateway 挂双通道。** 在 `openclaw.json` 的 `channels` 下同时配置 `telegram` 和 `discord`，各自 token + 白名单，指向同一个 agent workspace：

```json
{
  "channels": {
    "telegram": { "botToken": "***", "dmPolicy": "allowlist" },
    "discord":  { "botToken": "***" }
  },
  "agents": { "default": { "workspace": "~/.openclaw/workspace" } }
}
```

**第二步：session key 里显式带平台。** 路由层按 `platform:chatId` 隔离短期上下文，agent 级记忆共享。这一步是上下文串线的第一道闸。

**第三步：入站归一化。** 写一个 hook，把所有平台消息拍平成统一结构：

```js
export function onMessage(msg) {
  return {
    platform: msg.channel,               // "telegram" | "discord"
    sessionKey: `${msg.channel}:${msg.peerId}`,
    text: stripMentions(msg.text),
    attachments: normalizeFiles(msg.attachments),
  };
}
```

**第四步：出站适配。** 发送前按平台转格式：Telegram 用 HTML 模式只转义 `< > &`；Discord 截断到 2000 字符，超长转成文件附件。

**第五步：触发规则。** Discord 频道只响应 @ 和 reply；TG 群响应 /命令 或 @；两边私聊全响应。

## 踩坑点

- **MarkdownV2 转义地狱**：TG 的 MarkdownV2 要转义的字符多达十几个，换成 HTML 模式后 400 错误基本消失；
- **session key 漏了 platform**：上线第一天两平台上下文就串了，key 必须含平台标识；
- **Discord 附件 URL 会过期**：记忆里存 URL 是坑，附件落地到本地存储再引用；
- **限流**：Discord 同频道 5 条/5 秒，TG 群有每群分钟级限制——合并连发消息，宁可延迟一秒也别触发 429；
- **长任务要占位**：agent 干活超过 10 秒先回一句"处理中"或 typing 状态，否则用户会重复触发；
- **幂等**：gateway 重启后 polling 会重推消息，按 message id 去重。

## 可复用建议

1. **入站归一化、出站适配**，中间的路由和 agent 逻辑只处理平台无关的数据；
2. 渠道层保持薄，逻辑放 gateway hook/插件里，不要写死在 system prompt；
3. 准备一组"格式酷刑"测试消息：代码块、emoji、嵌套引号、@全体、超长文本，上线前全过一遍；
4. 日志统一带 `[tg]` / `[dc]` 前缀，排障效率提升明显。

## 总结

双通道接入的配置量其实很小，真正的工程量在**消息边界上的归一化与适配**。把平台差异挡在渠道层，agent 层保持干净，之后再加 Slack 或飞书，也只是多写一个适配器的事。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-24/9ebd44f9562ca906.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-24/b37cde9f09d6ebf1.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-24/acbf937f54d59fa6.png)

