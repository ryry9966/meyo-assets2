---
title: 一份大脑，两个入口：让同一个 Agent 同时值守 Telegram 和 Discord
feedId: 42652
source: 综合讨论
publishedAt: 2026-10-26
---

## 背景

Agent 在 OpenClaw 上跑通之后，第一个自然需求就是"到处都能找到它"。我的场景：同一个个人 agent，平时在 Telegram 上当私助，同时在几个 Discord 频道做答疑。最早偷懒，两个平台各跑一套 bot、各接一份 prompt，结果记忆不互通、技能重复维护、一边改了另一边忘同步。后来收敛成现在这套：**一个 gateway、一个 agent、两个 channel**。

## 问题

多渠道不是"多填一个 token"，真正的麻烦在四层：

1. **路由与身份**：不同平台的消息怎么映射到 session，谁有权限触发；
2. **格式差异**：Markdown 方言、长度上限（Discord 2000 字符 / Telegram 4096）、代码块与表格的渲染表现；
3. **噪声控制**：群聊里 bot 该是沉默的大多数，而不是刷屏机器；
4. **运维环境**：两个平台对网络出口的要求并不一样。

## 做法

**1. 单网关多渠道。** 在 `openclaw.json` 的 `channels` 里同时配置两端（字段名随版本有微调，以你版本的文档为准；token 走环境变量）：

```jsonc
{
  "channels": {
    "telegram": {
      "botToken": "${TELEGRAM_BOT_TOKEN}",
      "allowFrom": ["<你的tg数字id>"],
      "groups": { "-100…": { "requireMention": true } }
    },
    "discord": {
      "token": "${DISCORD_BOT_TOKEN}",
      "guilds": { "<guild>": { "channels": { "general": { "requireMention": true } } } }
    }
  }
}
```

**2. 一个 agent，不拆。** 两个渠道都路由到同一个默认 agent。OpenClaw 会按渠道加会话生成不同的 session key（如 `telegram:chatid`、`discord:channel:thread`），会话天然隔离；而 workspace 里的 `AGENTS.md`、记忆文件、skills 和 MCP 工具是共享的——这正是我们要的：对话分平台，大脑和工具只有一份。

**3. 权限收敛。** 群聊一律 `requireMention`，白名单只放行自己人和测试频道；先私聊跑通，再进群。

**4. 输出约束放配置层，不放 prompt 层。** 长回复由网关分段，Markdown 方言由网关转换；prompt 里只保留"回复尽量短、少用表格"这类跨端都成立的习惯。

## 踩坑点

- **Discord 收不到正文**：开发者后台没开 Message Content Intent，bot 只能看到空消息，排查了半小时才发现。
- **Telegram 群里"聋了"**：BotFather 的 privacy mode 默认开启，@ 之外的上下文全收不到；要么关掉，要么给 bot 群管理员身份。
- **超大代码块分段后碎了**：触发 Discord 2000 字符上限被切开，渲染成两截。让 agent 对长代码主动降级为"存文件+给摘要"，比事后硬切体面。
- **定时任务双响**：cron/心跳类技能没绑定渠道，两边同时播报。定时类输出固定走单一渠道。
- **Discord 线程会话碎片化**：论坛帖/thread 会产生新 session key，长期跑会攒一堆碎片会话，定期清理或做映射。
- **串台先查 session 映射**："在 A 群回答了 B 群的事"，九成是 session key 或引用回复逻辑的问题，别急着改 prompt。

## 可复用建议

- 能一个 agent 就别拆两个：记忆、技能、评测只有一份，回归成本减半；
- 平台差异留在网关层消化，prompt 保持渠道无关，将来加 Slack/飞书就是加配置，不是改大脑；
- 白名单 + mention 触发是群聊最省心的默认值，宁可慢一步接入，不要一上来全量监听；
- 配置进 git、token 走环境变量、gateway 放在网络出口干净的服务器上；
- 给路由留一条调试日志，能省掉之后大量"它为什么没回我"的排查。

## 总结

跨平台路由的本质不是"多接一个 bot"，而是把渠道差异在网关层归一化，让 Agent 层对消息来源无感知：会话隔离靠 session key，记忆共享靠 workspace，噪声与权限靠配置兜底。这套收敛完之后，再加第三个渠道，动的只是配置文件。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-26/82db53e70e0745d6.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-26/7769cbbde5a7d3fe.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-26/df088b0d7079da0b.png)

