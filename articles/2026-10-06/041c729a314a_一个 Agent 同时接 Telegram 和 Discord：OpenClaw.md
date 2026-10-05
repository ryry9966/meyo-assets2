---
title: 一个 Agent 同时接 Telegram 和 Discord：OpenClaw 跨平台消息路由实践
feedId: 40612
source: 综合讨论
publishedAt: 2026-10-06
---

## 背景

自己跑了一段时间 OpenClaw 后，入口分散在两个平台：Telegram 放私聊提醒和随手问答，Discord 挂在社群频道里。最初图省事起了两个 gateway、两个 bot token，结果很快出现上下文不同步、token 管理混乱的问题。目标于是变成：**一个 Agent、一份记忆，两个平台同时服务**，而不是部署两套。

## 问题

“能不能接”不是难点，真正的工程问题有三个：

1. **会话隔离**：两边消息必须进不同 session，避免上下文串台；
2. **记忆共享**：跨平台对话要能延续同一份 workspace 和 memory；
3. **平台差异**：消息长度上限（Telegram 4096 / Discord 2000）、Markdown 方言、流式输出行为、频控各不相同。

## 做法

1. **准备两个凭证**：BotFather 拿 Telegram bot token；Discord 开发者后台建应用、开启 Message Content Intent，用 OAuth2 链接邀请进服务器。
2. **同时启用两个 channel**：在 `~/.openclaw/openclaw.json` 里配置（示意）：

```json
{
  "channels": {
    "telegram": { "botToken": "..." },
    "discord":  { "token": "..." }
  }
}
```

   两侧都配好 allowlist，只放行自己的账号和服务器。
3. **会话策略保持默认**：session key 通常是 `channel:chatId`，天然按聊天隔离，不需要手动合并。
4. **记忆共享靠同一个 Agent 实例**：两个 channel 的流量都进同一个 gateway、同一个 agent runtime，读写的都是同一份 workspace，跨平台上下文自然延续。
5. **格式适配交给网关层**：markdown 转换、长消息分段由 channel 适配层处理；Discord 侧建议关闭或降频“编辑消息式”流式输出。
6. **验证**：重启 gateway，两边各发一条消息，翻日志确认 session key 与路由是否符合预期。

进阶玩法：用 agents + bindings 把 Discord 的某个频道绑到另一个 agent（不同人格或模型），Telegram 继续走主 agent。

## 踩坑点

- **Telegram 409 Conflict**：两个进程同时 poll 同一个 bot 会冲突，确认只有一个 gateway 在跑。
- **Discord 2000 字符上限**：长回复必须分段发送，别指望客户端。
- **MarkdownV2 转义**：代码块里的 `_` `*` 很容易触发 Telegram 400，转义要在适配层统一做。
- **Discord 限速**：流式编辑消息太频繁会被限速，表现为回复“卡住”，降频即解决。
- **Intent 没开**：Discord 收到的消息正文为空，Agent 拿到“空气”，且不报错——新手最容易漏。
- **群组策略**：设成“被提及才回”，避免两个平台互相转发造成自激循环。

## 可复用建议

- 排查路由问题的第一步永远是看日志里的 **session key**，答案几乎都在 key 里。
- 平台差异全部收在 channel 适配层，Agent 侧只管对话逻辑，别在 prompt 里写平台特判。
- allowlist 一开始就收紧：公网 bot 的第一原则是少接陌生人。
- 用 systemd 或 pm2 托管 gateway 并自动重启，长轮询断线是常态而非异常。

## 总结

跨平台路由不是“多接一个插件”，本质是**会话模型 + 记忆共享 + 平台适配**三件事。OpenClaw 默认的 session 隔离加共享 workspace 已经解决了前两件的大头，剩下的是频控和转义这类脏活。按上面的顺序配，半小时内能跑通；之后要再加 Slack 之类的入口，模式完全一样。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-06/74fc11505cacaa63.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-06/5df13a0cdfd705bf.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-06/956fad2b8ed5803a.png)

